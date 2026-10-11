# Lições aprendidas

[← README](../../README.md) · [Runbook](runbook.md) · [Reverter](rollback.md)

O resumo dos aprendizados está no [ROADMAP.md](../../ROADMAP.md). Aqui fica o caminho da investigação, o que não saiu como esperado e o que faríamos diferente.

## O caminho da investigação

1. **Servidor primeiro.** Hora e fuso, qual serviço de hora está rodando, fontes e desvio (`chronyc`), versão e caminhos do Samba, `smb.conf` e permissões do socket. Resultado: o servidor estava certo, só não atendia os clientes.
2. **Rede.** `w32tm /stripchart` de uma estação até o servidor: ele responde? qual é o desvio? Isso mostra se a UDP 123 passa e quantifica o problema.
3. **Cliente.** `w32tm /query /source` e `/status` (fonte e última sincronização), `w32tm /query /configuration` (tipo e fontes) e `nltest /dsgetdc:... /TIMESERV` (o domínio anuncia um servidor de hora?). Resultado: configuração certa, mas sem sincronizar.
4. **Hipótese e teste mínimo.** "Os clientes só não tentaram de novo depois que o servidor passou a responder assinado." Confirmado com `w32tm /resync /rediscover` em **uma** estação.
5. **Só então a GPO**, em fases.

## O que não saiu como esperado

- **Caminho genérico, instalação específica.** A receita inicial usava `/var/lib/samba/ntp_signd`. No Samba compilado do código-fonte o caminho era `/usr/local/samba/var/lib/ntp_signd`. A investigação do servidor descobriu isso antes de qualquer alteração.
- **Comandos e conteúdo de arquivo no mesmo bloco.** As linhas `allow` e `ntpsigndsocket` foram coladas no terminal e deram `command not found`. Sem efeito colateral, mas mostrou que cada bloco precisa dizer se é comando ou conteúdo de arquivo.
- **`SpecialPollInterval` para acelerar a correção.** A ideia inicial era reduzi-lo, mas no modo `NT5DS` ele não é usado. Foi retirada antes de ser aplicada.
- **Esperar um salto de hora.** Depois do `resync`, a diferença continuou quase igual. Era a correção gradual do Windows: só a medição depois de cerca de 30 minutos mostrou o resultado.
- **Consulta no caminho errado do registro.** Dois valores da GPO (`NtpServer` e `Type`) estão em `...\W32time\Parameters`, e não em `...\TimeProviders\NtpClient`.
- **`gpresult` ambíguo.** O nome da GPO aparece também na lista de filtradas. Passamos a confirmar pelos valores no registro.

## Decisões de risco que valeram a pena

- Restringir o alcance (filtro de segurança em uma estação) **antes** de vincular a GPO.
- Reiniciar o serviço Samba e o servidor para ver se a permissão do diretório do socket persiste.
- Fazer backup antes de editar e ter um comando de reversão para cada alteração.
- Medir antes e depois com os mesmos comandos.

## O que faríamos diferente

- Registrar as medições (valor e hora) em uma tabela desde o início.
- Medir também a estação sobre a qual alguém disse "já está certa", antes de dar o assunto por encerrado.
- Preparar cedo um script de verificação (está no [roadmap](../../ROADMAP.md)).
