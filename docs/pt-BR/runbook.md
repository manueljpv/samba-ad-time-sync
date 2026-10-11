# Runbook: sincronização de hora em um domínio Samba AD

[← README](../../README.md) · [Reverter](rollback.md) · [Lições aprendidas](lessons-learned.md)

> **Aviso: dados fictícios.** Todos os nomes, endereços e valores deste documento são fictícios (domínio `example.local`, faixas reservadas para documentação pela RFC 5737). Adapte ao seu ambiente antes de usar, e faça cada alteração com backup e uma estação de teste.

## 1. Cenário

- Domínio `example.local`, com um controlador `dc01.example.local` (`192.0.2.10`): Samba AD DC compilado do código-fonte em `/usr/local/samba`, Rocky Linux 10.x, máquina virtual (KVM).
- Estações Windows em OUs de computadores organizadas por departamento.
- Redes que consultam o servidor de hora: `192.0.2.0/24` (rede principal, com servidores e algumas estações), `198.51.100.0/24` e `203.0.113.0/24` (redes de estações). As redes de celulares, visitantes e IoT não têm máquinas do domínio e **não foram liberadas**, de propósito.

## 2. Problema

Dispositivos do domínio com a hora divergente em minutos: alguns adiantados, outros atrasados.

## 3. Diagnóstico

Faça o diagnóstico antes de alterar qualquer coisa, e anote os valores.

**No servidor** (somente leitura):

```bash
date; timedatectl                         # hora e fuso
chronyc tracking; chronyc sources -v      # sincronização e fontes
grep -v -E '^\s*(#|$)' /etc/chrony.conf   # configuração em uso
samba -b | grep -i NTP_SIGND_SOCKET_DIR   # caminho do socket de assinatura
ls -ld /caminho/do/ntp_signd              # permissões do diretório
```

**Em uma estação Windows** (PowerShell como administrador, somente leitura):

```
w32tm /query /source
w32tm /query /status
w32tm /stripchart /computer:dc01.example.local /samples:3 /dataonly
w32tm /query /configuration
nltest /dsgetdc:example.local /TIMESERV
```

O que encontramos:

- O servidor `dc01` estava com a hora correta (diferença de cerca de 0,1 ms contra servidores NTP públicos) e com o `chronyd` ativo, **mas sem atender clientes**: o `chrony.conf` não tinha `allow` nem `ntpsigndsocket`.
- Estações medidas antes da correção:

| Estação | Diferença para o servidor | Fonte de hora | Última sincronização |
|---|---|---|---|
| WS-B | cerca de 1,75 s | Local CMOS Clock | nunca |
| WS-C | cerca de 11,2 s | Local CMOS Clock | nunca |
| WS-A | cerca de 128 s (2 min 8 s) | Local CMOS Clock | nunca |

- A configuração dos clientes já estava correta (`Type: NT5DS`, cliente NTP habilitado), e o domínio já anunciava o `dc01` como servidor de hora (sinalizadores `TIMESERV` e `GTIMESERV` no `nltest`).
- **Causa:** o servidor não respondia hora **assinada** (MS-SNTP) aos clientes Windows. Sem resposta válida, cada relógio seguia o próprio hardware e foi se desviando.
- O caminho UDP 123 entre as redes foi validado com `w32tm /stripchart`.

## 4. Alterações no servidor (dc01)

Todas reversíveis (veja [rollback.md](rollback.md)). O Samba não precisou de nenhuma opção nova no `smb.conf`. Faça um passo por vez e confira o resultado antes do seguinte.

**Passo 1. Dar ao usuário `chrony` acesso ao diretório do socket de assinatura** (comandos de shell):

```bash
chgrp chrony /usr/local/samba/var/lib/ntp_signd
chmod 750 /usr/local/samba/var/lib/ntp_signd
```

Resultado esperado: `drwxr-x--- root chrony`. O socket dentro dele continua `root:root srwxrwxrwx`.

**Passo 2. Backup da configuração do chrony** (comando de shell):

```bash
cp -a /etc/chrony.conf /etc/chrony.conf.bak-YYYYMMDD
```

**Passo 3. Acrescentar ao final do `/etc/chrony.conf`** as linhas de [configs/chrony.conf.example](../../configs/chrony.conf.example). Elas são **conteúdo de arquivo**, não comandos: não as digite no terminal. As linhas originais ficam intactas.

```
# Servidor de hora do dominio example.local
allow 192.0.2.0/24
allow 198.51.100.0/24
allow 203.0.113.0/24
ntpsigndsocket /usr/local/samba/var/lib/ntp_signd
```

**Passo 4. Reiniciar o `chronyd`** (comando de shell):

```bash
systemctl restart chronyd
```

O log passa a mostrar `MS-SNTP authentication enabled` e o `chronyd` começa a escutar na UDP 123. Logo após o reinício, o `chronyc tracking` pode mostrar `Not synchronized` por um ou dois minutos: espere antes de testar qualquer cliente.

**Observação sobre rede e segurança:** em outros ambientes, confirme que a UDP 123 está liberada no firewall do servidor e que o SELinux (se ativo) permite que o `chronyd` leia o diretório do socket. Mantenha o `allow` restrito às redes que realmente têm máquinas do domínio.

## 5. Validações no servidor

- `chronyc tracking`: `Leap status: Normal`, estrato 2, diferença de microssegundos a poucos milissegundos.
- **Reinício do serviço `samba`:** a permissão do diretório `ntp_signd` persistiu (`root:chrony`), o socket foi recriado e as portas 53, 88, 389 e 445 voltaram a escutar.
- **Reinício do servidor:** `samba` e `chronyd` subiram sozinhos, a permissão persistiu e o chrony sincronizou.
- Nos dois casos, a estação de teste sincronizou com `Sinalizadores de Fonte de Tempo: 2 (Autenticado)` e `Último Erro de Sincronização: 0` (no seu Windows, os nomes aparecem no idioma do sistema).

Os reinícios afetam o domínio por alguns segundos (serviço Samba) ou minutos (servidor): login, DNS interno e GPO ficam indisponíveis. Faça em um horário de pouco movimento.

## 6. Resultado nos clientes

- **WS-C** (estação de teste): `w32tm /resync /rediscover` mudou a fonte para `dc01.example.local`. Para diferenças pequenas o Windows corrige o relógio de forma **gradual** (slew), e não com um salto: a diferença de 11,2 s caiu para cerca de 10 ms em aproximadamente 30 minutos.
- **WS-B** e outra estação (WS-D) se corrigiram **sozinhas**, sem nenhum comando, depois que o servidor passou a responder de forma assinada.
- **WS-A** (a que estava cerca de 128 s fora): o usuário relatou que o relógio já estava correto. **Medição técnica ainda pendente.**

## 7. GPO "Time Sync - NT5DS"

Criada para uniformizar a configuração dos clientes. Ela é a **segunda camada**: a primeira é o servidor saudável, e nenhuma GPO substitui isso.

Configuração do Computador > Políticas > Modelos Administrativos > Sistema > Serviço de Hora do Windows > Provedores de Hora:

- **Habilitar Windows NTP Client:** Habilitado.
- **Configurar Windows NTP Client:** Habilitado, com `NtpServer = dc01.example.local,0x9` e `Type = NT5DS`. Os demais campos ficam nos padrões da política (CrossSiteSyncFlags 2, ResolvePeerBackoffMinutes 15, ResolvePeerBackoffMaxTimes 7, SpecialPollInterval 1024, EventLogFlags 0).
- **Habilitar Windows NTP Server:** Não configurada (as estações não devem servir hora).

Onde os valores aparecem no registro das estações: `HKLM\SOFTWARE\Policies\Microsoft\W32time\Parameters` (`NtpServer` e `Type`) e `HKLM\SOFTWARE\Policies\Microsoft\W32time\TimeProviders\NtpClient` (`Enabled` e os demais).

### Escopo e fases

Para limitar o risco, a GPO foi aplicada em fases:

1. GPO criada **sem vínculo**, só com as políticas configuradas.
2. Filtro de segurança restrito a **uma estação de teste**, e só então o vínculo na raiz do domínio (não imposto).
3. Validação na estação de teste (`gpupdate /force`, `gpresult`, `w32tm /query /configuration`, `reg query`).
4. Filtro WMI `Workstations only (Windows Client)`, para não atingir servidores, e nova validação.
5. Filtro de segurança ampliado para `Domain Computers`, e validação em uma segunda estação.

Como o vínculo está na raiz, a herança cobre todas as OUs de computadores. As estações recebem a GPO no ciclo normal de atualização (90 a 120 minutos), sem reiniciar.

## 8. Observações técnicas

- No modo `NT5DS`, o `SpecialPollInterval` **não é usado**. O intervalo de 1024 s vem de `MinPollInterval` e `MaxPollInterval`. Esses parâmetros não foram alterados: a política "Global Configuration Settings" define muitos valores de uma vez e é arriscada.
- Os valores `NtpServer` e `Type` da política ficam em `...\W32time\Parameters`, e não em `...\TimeProviders\NtpClient`. Consultar o caminho errado dá a impressão de que a GPO não chegou.
- Verificar só o nome da GPO no `gpresult` não basta: ele aparece tanto na lista de aplicadas quanto na de filtradas. Confirme pelos valores no registro depois de um `gpupdate /force`.
- O serviço de hora do Windows vem com `StartType` `Manual` (início por gatilho), que é o padrão.

## 9. Pendências

1. **WS-A:** medir a diferença e a fonte de hora com `w32tm /query /source` e `w32tm /stripchart /computer:dc01.example.local /samples:3 /dataonly`. Hoje existe apenas o relato do usuário.
2. Conferir uma estação de cada rede de estações, em OUs diferentes, depois do ciclo de atualização das GPOs.
3. Verificar se alguma OU de computadores tem a herança de GPOs bloqueada (ícone de exclamação no GPMC).
