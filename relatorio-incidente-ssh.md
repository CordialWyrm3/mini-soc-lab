# INC-001: Tentativas de login SSH com usuário inexistente

| Campo | Valor |
|---|---|
| ID | INC-001 |
| Tipo | Simulação em laboratório (não é um incidente real) |
| Data | 04/11/2026, entre 16:31 e 16:33 (horário de Brasília, UTC-3) |
| Responsável | Brenno Matias |
| Sistema afetado | `cordi-server` (Ubuntu Server 26.04.1 LTS, servidor do Wazuh) |
| Ferramenta de detecção | Wazuh 4.14.9 |
| Gravidade observada | Nível 5 (média-baixa) |
| Status | Em andamento: falta o alerta de força bruta (nível 10) |

## 1. Resumo

Foram feitas tentativas de login SSH com um usuário que não existe (`usuarioinexistente`) contra o servidor do laboratório. O Wazuh detectou cada tentativa como um alerta de nível 5 (regra 5710). Como o total de tentativas ficou abaixo do limite da regra de força bruta, esse alerta de nível 10 não foi disparado.

## 2. Ambiente

- Servidor monitorado: `cordi-server`, com o agente embutido do Wazuh manager.
- Acesso SSH a partir do anfitrião Windows, pelo redirecionamento de porta 2222→22 do VirtualBox.
- Rede isolada, sem exposição à internet.

## 3. Cenário simulado

Execução manual, a partir do PowerShell do Windows, de `ssh -p 2222 usuarioinexistente@localhost`, digitando senhas incorretas a cada pedido. A simulação foi feita apenas contra o servidor do próprio laboratório.

## 4. Linha do tempo (horário de Brasília)

| Horário | Evento | Regra | Nível |
|---|---|---|---|
| 16:31:28 | Login SSH bem-sucedido (acesso de administração) | 5715 | 3 |
| 16:32:13 | Login SSH bem-sucedido (acesso de administração) | 5715 | 3 |
| 16:32:35 | Tentativa de login com usuário inexistente | 5710 | 5 |
| 16:32:39 | Tentativa de login com usuário inexistente | 5710 | 5 |
| 16:32:49 | Tentativa de login com usuário inexistente | 5710 | 5 |
| 16:33:01 | Tentativa de login com usuário inexistente | 5710 | 5 |
| 16:33:03 | Tentativa de login com usuário inexistente | 5710 | 5 |

## 5. Detecção

- **Regra 5710** (nível 5): "sshd: Attempt to login using a non-existent user". Disparou 5 vezes em cerca de 28 segundos.
- **Regra 5715** (nível 3): "sshd: authentication success". São os logins legítimos do próprio administrador, que aparecem na mesma consulta.
- Técnica MITRE ATT&CK associada às tentativas: **[PREENCHER com o valor de `rule.mitre.id` do alerta expandido]**.

## 6. Análise

**Legítimo ou suspeito.** Os dois alertas de nível 3 são ações do administrador. Os cinco de nível 5 foram gerados de propósito nesta simulação. Num ambiente real, um padrão assim (vários usuários inexistentes em sequência) sugere adivinhação de nomes de usuário.

**Por que não houve alerta de força bruta.** A regra de força bruta para usuário inexistente (5712, nível 10, técnica T1110) exige 8 ocorrências da regra 5710 em 120 segundos vindas do mesmo IP de origem. Foram 5 em cerca de 28 segundos, abaixo do limite. A regra equivalente para senha errada de usuário existente (5763) segue a mesma lógica de 8 em 120 segundos.

**Origem das tentativas.** O IP de origem registrado é **[PREENCHER com `data.srcip`; a expectativa é 10.0.2.2, o gateway do NAT do VirtualBox]**. Esse endereço não é o do cliente real, e sim o do ponto de passagem do NAT. Num caso real, esse achado exigiria investigar o que está entre o atacante e o servidor.

**Proteção do SSH.** O servidor tem `PerSourcePenalties` ativo no OpenSSH (por exemplo, `authfail:5`, `noauth:1`, `max:600`), que passa a recusar conexões de um IP que acumula falhas. Uma rajada posterior de conexões a partir do Windows foi recusada antes da autenticação, mas a causa exata não foi confirmada nos logs, então isso fica como hipótese.

## 7. Recomendações

1. Desativar o login por senha no SSH e exigir autenticação por chave (`PasswordAuthentication no`).
2. Reduzir `MaxAuthTries` e restringir quais usuários podem entrar por SSH (`AllowUsers`).
3. Limitar o acesso à porta 22 por firewall, permitindo só origens conhecidas.
4. Habilitar resposta ativa (por exemplo, o fail2ban ou a resposta ativa do Wazuh) para bloquear IPs após falhas repetidas, testando antes para não bloquear o próprio acesso de administração.
5. Criar uma regra ou um painel específico para repetições de usuário inexistente em janelas curtas.

## 8. Lições aprendidas e limitações

- A regra de força bruta tem limiar (frequência e janela de tempo). Entender esse limiar explica por que 5 tentativas não bastam.
- Atrás de NAT, o IP de origem perde valor para investigação.
- Os números do módulo de vulnerabilidades neste ambiente não são confiáveis, por causa do kernel muito recente.
- O desempenho do laboratório oscilou (congelamentos de CPU e painel lento), o que limitou o volume de testes.

## 9. Pendências para fechar o INC-001

- [ ] Repetir a simulação com 8 ou mais tentativas em menos de 120 segundos (por exemplo, a partir de dentro da VM) e capturar o alerta 5712.
- [ ] Preencher a técnica MITRE e o IP de origem com os valores do alerta expandido.
- [ ] Anexar os prints dos alertas (sem senhas nem dados sensíveis).

## 10. Evidências

- `evidencias/inc-001-eventos.png`: tabela de eventos no Wazuh (busca `agent.name: cordi-server and rule.groups: sshd`)
- `evidencias/inc-001-alerta-expandido.png`: detalhe de um alerta 5710, com `rule.mitre.id`, `data.srcip` e `full_log`
- `evidencias/inc-001-regra-5712.png`: definição da regra de força bruta (`frequency="8"`, `timeframe="120"`)
