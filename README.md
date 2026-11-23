# mini-soc-lab

Laboratório de monitoramento e resposta a incidentes para uma empresa fictícia de pequeno porte, montado com ferramentas open source em máquinas virtuais. O objetivo é praticar o ciclo completo de um analista: levantar requisitos, desenhar a arquitetura, implantar a detecção, simular incidentes em ambiente isolado e documentar cada um.

> **Status:** em andamento. A base (servidor SIEM, rede isolada, agente e primeiros alertas) está funcionando. Os cenários de incidente estão sendo documentados um a um.

## Cenário

Uma empresa fictícia com poucos servidores e estações, sem nenhum monitoramento de segurança. A proposta é implantar um SIEM/XDR, validar que ele enxerga eventos relevantes e produzir relatórios de incidente que um time de TI conseguiria usar.

## Arquitetura

```
                 Windows (anfitrião)
        https://localhost:8443   ssh -p 2222 / -p 2223
                        |
          NAT + redirecionamento de portas (VirtualBox)
                        |
   +--------------------+--------------------+
   |                                         |
 Cordi-server (Wazuh)                  alvo01 (agente Wazuh)
 Ubuntu Server 26.04.1 LTS             Ubuntu Server 26.04.1 LTS
 192.168.100.10                        192.168.100.20
   |                                         |
   +------------ rede interna "labnet" ------+
                 192.168.100.0/24
```

| Componente | Detalhe |
|---|---|
| Hipervisor | Oracle VirtualBox 7.2.20, anfitrião Windows |
| Sistema das VMs | Ubuntu Server 26.04.1 LTS |
| SIEM/XDR | Wazuh 4.14.9, instalação all-in-one (manager, indexer, dashboard e Filebeat) |
| Servidor | 6 GB de RAM, disco virtual de 40 GB |
| Alvo | 2 GB de RAM, agente Wazuh conectado ao servidor |
| Rede | Adaptador 1 em NAT com redirecionamento de portas (2222→22, 2223→22, 8443→443); adaptador 2 em rede interna `labnet` |

## O que já funciona

- [x] Servidor Wazuh instalado e painel acessível pelo navegador do anfitrião
- [x] Rede interna isolada entre servidor e alvo
- [x] VM-alvo criada por clone, com identidade própria (hostname, machine-id e chaves SSH novas)
- [x] Agente instalado no alvo e aparecendo como ativo no painel
- [x] Detecção de tentativas de login SSH com usuário inexistente, mapeadas no MITRE ATT&CK
- [ ] Alerta de força bruta (regra de nível 10) disparado e documentado
- [ ] Monitoramento de integridade de arquivos (FIM)
- [ ] Arquivo de teste EICAR
- [ ] Varredura de portas com Nmap a partir de uma máquina atacante em `labnet`

## Relatórios de incidente

| ID | Cenário | Status |
|---|---|---|
| [INC-001](relatorio-incidente-ssh.md) | Tentativas de login SSH com usuário inexistente | Em andamento |

## Aprendizados e problemas resolvidos

| Problema | Causa | Solução |
|---|---|---|
| O instalador do Wazuh desfazia a instalação ao iniciar o indexer | O systemd encerrou o serviço por timeout de inicialização (cerca de 6 minutos) | Criar um override do systemd com `TimeoutStartSec` maior antes de reinstalar |
| O instalador reclamava de memória | O Ubuntu enxergava menos de 4 GB dos 4 GB configurados | Aumentar a RAM da VM para 6 GB |
| A rede host-only do VirtualBox não podia ser criada | Driver de rede host-only ausente no Windows | Usar redirecionamento de portas no NAT e uma rede interna entre as VMs |
| Duas VMs idênticas confundiam o Wazuh | O clone herdou hostname, machine-id e chaves SSH | Regenerar as identidades no clone |
| Congelamentos (soft lockup) e painel lento | Hipótese: o VirtualBox rodando sobre o hipervisor do Windows, somado ao limite de RAM do anfitrião | Reduzir a carga (uma VM por vez) e registrar como limitação do ambiente |

## Limitações conhecidas

- O Ubuntu 26.04 não consta na lista de sistemas recomendados do instalador do Wazuh 4.14. A instalação funcionou, mas não é um ambiente oficialmente suportado.
- O módulo de detecção de vulnerabilidades reportou milhares de achados em um único pacote de kernel recém-instalado. Os números parecem inflados (falsos positivos prováveis para um kernel tão novo) e não devem ser lidos como risco real.
- O IP de origem dos alertas de SSH aparece como o gateway do NAT do VirtualBox, e não como o IP real de quem tenta o acesso. Num ambiente real, isso levaria a investigar proxies ou NAT no caminho.
- O hardware do anfitrião limita o laboratório a poucas VMs ligadas ao mesmo tempo.

## Como reproduzir (resumo)

1. Instalar o VirtualBox e criar uma VM com Ubuntu Server (4 GB de RAM no mínimo, 6 GB recomendados; 2 CPUs; 40 GB de disco).
2. Adicionar regras de redirecionamento de portas no adaptador NAT (SSH e painel).
3. Instalar o Wazuh seguindo o guia oficial de início rápido (instalação all-in-one): https://documentation.wazuh.com
4. Criar a rede interna `labnet` e clonar a VM limpa para o alvo; trocar hostname, machine-id e chaves SSH do clone.
5. Instalar o agente no alvo com o assistente **Deploy new agent** do painel, apontando para o IP do servidor na `labnet`.

> Nunca publique senhas, tokens ou prints que mostrem credenciais. As senhas geradas pelo instalador do Wazuh ficam fora deste repositório.

## Próximos passos

- Fechar o INC-001 com o alerta de força bruta
- Documentar um cenário de integridade de arquivos e outro com o arquivo EICAR
- Escrever o documento de requisitos e a análise de riscos da empresa fictícia
- Adicionar um diagrama de rede em imagem e prints dos alertas

## Autor

Brenno Matias, GitHub: https://github.com/CordialWyrm3
