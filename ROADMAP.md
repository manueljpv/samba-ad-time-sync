# Roadmap

**Português** · [English](ROADMAP.en.md) · [Español (Honduras)](ROADMAP.es-HN.md)

Melhorias previstas para o projeto e o que aprendemos até aqui. São ideias, sem prazo garantido, e a ordem não é definitiva.

## Curto prazo

Melhorias pequenas, que cabem no que já existe.

- [ ] **Fechar as pendências de validação.** Medir a WS-A com `w32tm /query /source` e `w32tm /stripchart` (hoje só existe o relato do usuário), conferir uma estação de cada rede de estações em OUs diferentes depois do ciclo de atualização das GPOs, e verificar se alguma OU de computadores tem a herança de GPOs bloqueada.
- [ ] **Script de verificação do servidor.** Reunir em um só script os comandos de validação (chrony sincronizado, assinatura habilitada, UDP 123 escutando, grupo e modo do diretório `ntp_signd`) e devolver "ok" ou "falha" com o motivo. Só entra no repositório depois de testado como script.
- [ ] **Alerta de desvio.** Monitorar o `Leap status` e o desvio do `chronyc tracking`, e a permissão do diretório do socket, com aviso se algo mudar. Sem isso, um problema só aparece quando alguém nota a hora errada.
- [ ] **Tempo de correção.** Avaliar, com cuidado, os parâmetros de sondagem do `w32time` (`MinPollInterval` e `MaxPollInterval`) para estações que demoram a se corrigir. Não foram alterados porque a política que os define mexe em muitos valores de uma vez.
- [ ] **GPO por script.** Avaliar criar e configurar a GPO por script (módulo `GroupPolicy` do PowerShell) contra um Samba AD, para a implantação ser repetível. Precisa de teste, porque nem todos os recursos de GPO se comportam igual com o Samba.
- [ ] **Verificação automática da documentação.** Checar links e formatação dos arquivos Markdown a cada alteração (GitHub Actions).

## Longo prazo

Mudanças maiores, que exigem projeto próprio.

- [ ] **Redundância do servidor de hora.** Hoje há um único servidor de hora. Avaliar um segundo controlador com chrony, e uma fonte alternativa para quando a internet cair (por exemplo, um relógio de referência local), pesando o risco de servir uma hora sem referência externa.
- [ ] **Equipamentos fora do domínio.** Câmeras, impressoras e celulares ficaram de fora de propósito. Avaliar atendê-los de forma controlada, por rede, com regras de firewall e sem ampliar o `allow` além do necessário.
- [ ] **Histórico e painel.** Registrar o desvio do servidor e das estações ao longo do tempo (por exemplo, com Prometheus e Grafana, ou Zabbix) para enxergar tendências e estações problemáticas.
- [ ] **Verificação em massa da frota.** Um relatório que liste a fonte e o desvio de cada estação sem depender de abrir máquina por máquina. O mecanismo ainda está por definir (por exemplo, um script de inicialização por GPO que grave o resultado em um compartilhamento).
- [ ] **Implantação com Ansible.** Um playbook idempotente que aplique a permissão do diretório, o `chrony.conf` e o reinício do serviço em um ou mais controladores Samba AD.
- [ ] **Vários sites.** Adaptar o desenho para ambientes com mais de um site e mais de um controlador.

## O que aprendemos

- **O problema não estava onde parecia.** O servidor estava com a hora certa, e a causa era ele não responder hora assinada aos clientes. Medir antes de mexer evitou "consertar" a coisa errada.
- **Leia o servidor antes de aplicar uma receita.** Em um Samba compilado do código-fonte, os caminhos mudam (`/usr/local/samba`, e não `/var/lib/samba`). Confirme o caminho do socket com `samba -b | grep -i NTP_SIGND_SOCKET_DIR`.
- **Separe o que é comando do que é conteúdo de arquivo.** Colar no terminal linhas que pertencem ao `chrony.conf` gera erros confusos. Cada bloco deve dizer onde vai.
- **Um passo por vez, com backup e como desfazer.** Cada alteração foi pequena, validada e reversível antes da seguinte.
- **O Windows corrige a hora aos poucos.** Para diferenças pequenas ele acelera o relógio em vez de dar um salto: 11 s levaram cerca de 30 minutos. Meça de novo antes de concluir que falhou.
- **Confirme a GPO pelos valores, não pelo nome.** O `gpresult` mostra o nome da GPO até quando ela foi filtrada. Os valores `NtpServer` e `Type` ficam em `...\W32time\Parameters`.
- **No modo NT5DS, o `SpecialPollInterval` não vale.** O intervalo de sondagem vem de outros parâmetros.
- **Teste o reinício antes de confiar.** Reiniciar o serviço Samba e o servidor inteiro mostrou que a permissão do diretório do socket persiste.
- **Restrinja antes de vincular.** Na GPO, limite o alcance (filtro de segurança em uma estação) antes de criar o vínculo, e amplie por fases.
- **Relato não é medição.** Quando alguém diz que "a hora já está certa", registre como relato e meça depois.
- **O seguro morreu de velho.** As estações se corrigiram sozinhas depois de o servidor ser consertado, mas a GPO continua valendo como segunda camada para ninguém ficar sem fonte de hora.
