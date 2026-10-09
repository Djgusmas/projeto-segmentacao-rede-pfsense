# Projeto 3 — Laboratório de Segmentação de Rede (pfSense + Suricata) · README / Portefólio

Oct 9, 2026 · @The M0derator

## Resumo

Laboratório de segurança de rede construído para demonstrar, de ponta a ponta, **segmentação de rede, política de firewall por defeito-negar e deteção de intrusões (IDS)**. Uma firewall pfSense separa três redes isoladas (gestão, uso doméstico e laboratório), com regras explícitas entre elas e um IDS Suricata a inspecionar o segmento de laboratório. O ambiente é inteiramente virtualizado (VMware Workstation), e o isolamento e a deteção foram validados com testes documentados a partir de uma máquina atacante Kali Linux.

`pfSense CE 2.7.2` · `Suricata` · `VLAN/segmentação` · `VMware Workstation` · `Kali Linux` · `CIS Controls v8` · `NIST CSF`

## Objetivo e competências demonstradas

O objetivo foi projetar, implementar e **validar** um ambiente que contém o raio de impacto de um comprometimento: uma rede de laboratório onde se corre software malicioso não pode alcançar a rede de uso doméstico nem a de gestão, e todo o seu tráfego é vigiado por um IDS.

Competências aplicadas:

- Projeto e implementação de **segmentação de rede** (sub-redes isoladas por interface)
- **Firewall stateful** com política *deny-by-default* e aliases
- **Deteção de intrusões** com Suricata (rulesets ET Open) e **escrita de assinaturas personalizadas**
- Diagnóstico e resolução de problemas reais de rede (conflitos de IP, mapeamento de interfaces, DNS, afinação do IDS)
- Documentação técnica e **mapeamento a frameworks de controlo** (CIS Controls v8, NIST CSF)

## Arquitetura

Três redes virtuais isoladas, cada uma ligada a uma interface própria do pfSense. O tráfego entre segmentos só existe se uma regra o permitir — por omissão, está tudo bloqueado.

&#91;embedded content: Arquitetura · pfSense e três segmentos isolados (mapeamento real das redes virtuais)\]

**Mapeamento de segmentos:**

| Segmento | Sub-rede | Rede VMware | Interface pfSense | Conteúdo |
| --- | --- | --- | --- | --- |
| Gestão | 10.10.10.0/24 | VMnet15 | LAN | administração do pfSense |
| Família | 10.10.20.0/24 | VMnet10 | OPT1 | dispositivos de uso doméstico |
| Laboratório | 10.10.30.0/24 | VMnet16 | OPT2 | Kali (atacante) + alvos; vigiado pelo Suricata |
| WAN | — | NAT (VMnet8) | WAN | saída para a internet |

## Implementação

**Firewall — política *deny-by-default*.** As interfaces dos segmentos nascem sem regras (logo, bloqueiam tudo) e as permissões são explícitas. Foi criado um alias `RedesPrivadas` (as três sub-redes internas) para bloquear o tráfego entre segmentos numa só regra. Política aplicada:

| Origem | Destino | Ação |
| --- | --- | --- |
| Laboratório (OPT2) | gateway / DNS (10.10.30.1) | Permitir |
| Laboratório (OPT2) | `RedesPrivadas` (outras redes internas) | **Bloquear** |
| Laboratório (OPT2) | Internet | Permitir |
| Família (OPT1) | `RedesPrivadas` | Bloquear |
| Família (OPT1) | Internet | Permitir |
| Gestão (LAN) | qualquer | Permitir (rede de administração) |

Como as regras são avaliadas por ordem, o bloqueio às redes internas é lido **antes** da permissão de internet — o resultado é acesso à internet **sem** acesso lateral às outras redes.

**IDS — Suricata.** Instalado e ativado na interface do laboratório (OPT2), em modo de deteção (IDS, sem bloqueio). Rulesets **ET Open** com as categorias `emerging-scan` e `emerging-attack_response` ativadas, e uma assinatura personalizada (ver secção própria). O *hardware offloading* (checksum, TSO, LRO) foi desligado, como o Suricata exige para inspecionar corretamente.

## Validação (provas)

O isolamento e a deteção foram verificados a partir da VM Kali (`10.10.30.100`), dentro do segmento de laboratório.

**Prova 1 — Segmentação seletiva.** Três pings a partir do laboratório demonstram, numa só captura, que há internet mas não acesso lateral:

| Destino | Significado | Resultado |
| --- | --- | --- |
| `1.1.1.1` | Internet | 0% de perdas — **permitido** |
| `10.10.10.1` | Gestão | 100% de perdas — **bloqueado** |
| `10.10.20.1` | Família | 100% de perdas — **bloqueado** |

> *(inserir screenshot: `provas/01-isolamento-segmentos.png`)*

**Prova 2 — Deteção pelo IDS.** Tráfego gerado a partir da Kali foi detetado pelo Suricata na interface OPT2, com o alerta a registar origem, destino e a assinatura disparada.

> *(inserir screenshot: `provas/02-suricata-deteccao.png`)*

As duas provas, em conjunto, demonstram o ciclo completo: um ambiente de risco **contido** pela firewall e **observável** pelo IDS.

## Regra de deteção personalizada

As regras de scan do ET Open são afinadas para tráfego de **fora para dentro**; o tráfego de reconhecimento a sair do laboratório não lhes correspondia. Para validar que o motor de deteção estava operacional e inspecionava o segmento nos dois sentidos, foi escrita uma assinatura Suricata personalizada:

```
alert icmp any any -> any any (msg:"TESTE LAB - ICMP detetado"; sid:1000001; rev:1;)
```

Após recarregar o Suricata, um simples `ping` a partir da Kali passou a gerar o alerta, confirmando a inspeção em ambos os sentidos (pedido e resposta ICMP). Mais do que um teste, isto demonstra a capacidade de **escrever deteções à medida** — uma competência central de engenharia de deteção, raramente presente num portefólio júnior.

## Problemas encontrados e resolução

A parte mais valiosa do projeto não foi seguir passos, mas diagnosticar e resolver problemas reais — o trabalho de quem opera redes.

- **Arranque preso na ISO.** A VM reiniciava sempre para o instalador (estado "Amnesiac", `config.xml` a 0 bytes). Causa: a ISO continuava ligada no arranque. Resolvido desligando o *Connect at power on* do leitor virtual.
- **Interfaces trocadas.** A ordem dos adaptadores na VM não coincidia com a atribuição no pfSense, pelo que a interface de gestão saía num segmento errado. Diagnosticado comparando a ordem dos adaptadores (em0–em3) com o estado de cada link e reatribuindo as interfaces.
- **Conflito de IP na gestão.** O anfitrião e o pfSense reclamavam ambos o `10.10.10.1`; o browser ligava-se a si próprio e recusava (`connection refused`), apesar de o ping "responder". Resolvido atribuindo ao anfitrião um IP distinto (`10.10.10.50`) na rede de gestão.
- **Numeração das redes virtuais.** O VMware atribuiu VMnet15/10/16 em vez dos números planeados; confirmou-se o mapeamento real por sub-rede em vez de confiar no número.
- **DNS bloqueado pela segmentação.** A regra *deny-by-default* bloqueava também os pedidos DNS do laboratório ao gateway — efeito correto da política, resolvido com uma regra explícita a permitir a porta 53 ao gateway.
- **IDS "sem alertas".** O Suricata corria sem gerar alertas. A análise do `suricata.log` confirmou o motor saudável (1356 regras, captura em pcap); a causa era a ausência de assinaturas correspondentes ao tráfego de saída, resolvida com a regra personalizada. **Lição:** distinguir "o IDS não funciona" de "não há assinatura para este tráfego".

## Mapeamento a frameworks

Os controlos implementados, enquadrados nas normas que um analista de risco reconhece:

| Controlo implementado | CIS Controls v8 | NIST CSF |
| --- | --- | --- |
| Segmentação de rede | 12 (Network Infrastructure Mgmt) · 4 (Secure Config) | PR.AC / PR.PT |
| Firewall *deny-by-default* | 4 · 12 | PR.AC |
| IDS + registo de eventos (Suricata) | 13 (Network Monitoring & Defense) · 8 (Audit Logs) | DE.CM / DE.AE |
| Rede de gestão separada | 12 | PR.AC |

*(Referências ao nível da família de controlo, para enquadramento.)*

## Stack e próximos passos

**Tecnologias:** pfSense CE 2.7.2 · Suricata (ET Open) · VMware Workstation Pro · Kali Linux · nmap · FreeBSD.

**Próximos passos** (evoluções planeadas, cada uma mais uma entrada no portefólio):

- Adicionar uma VM **alvo vulnerável** (Metasploitable) no laboratório, para scans internos realistas que disparam as assinaturas ET Open naturalmente.
- Resolver o DNS do laboratório através do pfSense (DNS Resolver), em vez de DNS externo.
- Reencaminhar os logs do Suricata e da firewall para um **SIEM** (Wazuh/Elastic) com dashboards — liga ao projeto seguinte do portefólio.
- Montar um domínio **Active Directory** no segmento de laboratório, para cenários de ataque e deteção mais completos.
- Passar o Suricata a **IPS** (bloqueio inline) depois de afinadas as assinaturas.

## Para o CV

**Linhas prontas a adaptar** (escolher 2–3):

- Projetei e implementei um laboratório de rede virtualizado com **segmentação em três redes isoladas** e firewall pfSense com política *deny-by-default*, contendo um ambiente de risco face à restante rede.
- Implementei **deteção de intrusões com Suricata** (rulesets ET Open) e **escrevi assinaturas personalizadas**, validando o isolamento entre segmentos e a deteção de tráfego a partir de uma máquina Kali.
- Diagnostiquei e resolvi problemas reais de rede (conflitos de IP, mapeamento de interfaces, DNS, afinação do IDS) num ambiente pfSense/FreeBSD.
- Documentei os controlos implementados com **mapeamento a CIS Controls v8 e NIST CSF**.

**Competências (secção de skills):** Segmentação de rede · Firewall (pfSense) · IDS/IPS (Suricata) · Engenharia de deteção · Virtualização (VMware) · Kali Linux / nmap · Hardening e troubleshooting de rede · GRC (CIS, NIST).

---

*Sugestão de estrutura do repositório:* `README.md` (este documento) · `/diagramas` (a arquitetura) · `/provas` (as duas screenshots) · `/config` (excertos das regras de firewall e da assinatura Suricata).
