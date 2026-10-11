# Como reverter

[← README](../../README.md) · [Runbook](runbook.md) · [Lições aprendidas](lessons-learned.md)

Cada parte do projeto pode ser desfeita sozinha. A ordem recomendada é **primeiro a GPO e depois o servidor**: assim as estações nunca ficam apontadas para um servidor que deixou de responder de forma assinada.

> Dados fictícios: `dc01.example.local`, `example.local` e as faixas `192.0.2.0/24`, `198.51.100.0/24` e `203.0.113.0/24`.

## 1. GPO "Time Sync - NT5DS"

Escolha o nível de reversão, do mais leve ao mais completo:

| O que fazer | Efeito |
|---|---|
| Remover `Domain Computers` do filtro de segurança | A GPO volta a valer só para as máquinas que sobrarem no filtro (por exemplo, a estação de teste) |
| Desmarcar **Vínculo Habilitado** do link na raiz | A GPO deixa de valer para todas, mas continua existindo |
| Excluir o link e, se quiser, a própria GPO | Remoção definitiva |

Nas estações, os valores da política somem no próximo ciclo de atualização (90 a 120 minutos) ou com `gpupdate /force`, e passa a valer a configuração **local**, que já era `NT5DS`. Nenhuma configuração local foi alterada nas estações durante o projeto: tudo vinha da GPO.

Para confirmar na estação:

```
gpupdate /force
reg query "HKLM\SOFTWARE\Policies\Microsoft\W32time\Parameters"   # os valores de política devem ter sumido
w32tm /query /configuration                                       # Type: NT5DS (Local)
```

## 2. Servidor (dc01)

> **Atenção:** reverter o servidor devolve o problema original. As estações deixam de receber hora assinada e voltam a ficar sem fonte de hora do domínio.

Comandos de shell, no servidor:

```bash
cp -a /etc/chrony.conf.bak-YYYYMMDD /etc/chrony.conf    # restaura o backup feito antes da alteração
systemctl restart chronyd
chgrp root /usr/local/samba/var/lib/ntp_signd           # opcional: volta o grupo original do diretório
```

Para confirmar:

```bash
grep -v -E '^\s*(#|$)' /etc/chrony.conf    # sem allow nem ntpsigndsocket
ss -ulpn | grep ':123'                     # o chronyd deixa de escutar na UDP 123
ls -ld /usr/local/samba/var/lib/ntp_signd  # drwxr-x--- root root
```

O `smb.conf` e o Samba não foram alterados, então não há nada a desfazer neles e o serviço Samba não precisa ser reiniciado.

## 3. Sincronização manual em uma estação

O `w32tm /resync` não altera nenhuma configuração: apenas pede uma sincronização. Não há nada a reverter nesse passo.
