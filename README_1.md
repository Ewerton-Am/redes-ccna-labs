# Laboratório 01 — VLSM, VLANs e Roteamento Inter-VLAN

**Cenário:** *Amorim Tech Solutions* — projeto e implementação da rede da matriz, a partir de um único bloco `172.16.0.0/24`, usando VLSM para atender cinco segmentos sem desperdício de endereços.

**Ferramenta:** Cisco Packet Tracer
**Tópicos:** VLSM · VLANs (802.1Q) · Roteamento inter-VLAN (router-on-a-stick) · DHCP · Rota estática

---

## 1. Objetivo

Segmentar um bloco `/24` em cinco sub-redes de tamanhos diferentes via **VLSM**, isolar cada setor em sua própria **VLAN**, prover roteamento entre elas com **router-on-a-stick**, entregar endereços por **DHCP** e interligar dois roteadores por um link ponto-a-ponto com **rota estática** — validando a conectividade fim-a-fim.

---

## 2. Topologia

![Topologia da rede](img/topologia.png)

A rede é dividida em dois blocos, cada um com seu roteador (2911) e switch (2960), interligados por um link `/30`:

- **Router0 / Switch0** → VLAN 10 (Produção) e VLAN 20 (Administração)
- **Router1 / Switch1** → VLAN 30 (T.I.) e VLAN 40 (Servidor)

---

## 3. Plano de Endereçamento (VLSM)

Alocação do maior para o menor segmento, a partir de `172.16.0.0/24`:

| Setor | Hosts | Sub-rede | CIDR | Faixa de hosts | Broadcast | Gateway |
|-------|-------|----------|------|----------------|-----------|---------|
| Produção | 60 | `172.16.0.0` | /26 | .1 – .62 | .63 | .62 |
| Administração | 28 | `172.16.0.64` | /27 | .65 – .94 | .95 | .94 |
| T.I. | 12 | `172.16.0.96` | /28 | .97 – .110 | .111 | .108 |
| Servidor | 5 | `172.16.0.112` | /29 | .113 – .118 | .119 | .114 |
| Link P2P | 2 | `172.16.0.120` | /30 | .121 – .122 | .123 | — |

> Cada sub-rede começa exatamente onde a anterior termina, garantindo zero sobreposição.

---

## 4. Configurações-chave

### Router0 (R1) — Produção + Administração

```cisco
! Exclusão dos gateways do pool DHCP
ip dhcp excluded-address 172.16.0.62
ip dhcp excluded-address 172.16.0.94
!
ip dhcp pool PRODUCAO
 network 172.16.0.0 255.255.255.192
 default-router 172.16.0.62
 dns-server 8.8.8.8
ip dhcp pool ADMINISTRACAO
 network 172.16.0.64 255.255.255.224
 default-router 172.16.0.94
 dns-server 8.8.8.8
!
interface GigabitEthernet0/0
 ip address 172.16.0.121 255.255.255.252
!
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 172.16.0.62 255.255.255.192
interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 172.16.0.94 255.255.255.224
!
! Rotas estáticas para as redes atrás do R2
ip route 172.16.0.96 255.255.255.240 172.16.0.122
ip route 172.16.0.112 255.255.255.248 172.16.0.122
```

### Router1 (R2) — T.I. + Servidor

```cisco
interface GigabitEthernet0/0
 ip address 172.16.0.122 255.255.255.252
!
interface GigabitEthernet0/1.30
 encapsulation dot1Q 30
 ip address 172.16.0.108 255.255.255.240
interface GigabitEthernet0/1.40
 encapsulation dot1Q 40
 ip address 172.16.0.114 255.255.255.248
!
! Rotas estáticas para as redes atrás do R1
ip route 172.16.0.0  255.255.255.192 172.16.0.121
ip route 172.16.0.64 255.255.255.224 172.16.0.121
```

### Switches — VLANs e trunk

```cisco
! Switch0
vlan 10
 name PRODUCAO
vlan 20
 name ADMINISTRACAO
!
interface range FastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 10
interface range FastEthernet0/3 - 4
 switchport mode access
 switchport access vlan 20
!
interface GigabitEthernet0/1
 switchport mode trunk
```

---

## 5. Evidências de Teste

### DHCP entregando endereços dinamicamente

![show ip dhcp binding](img/dhcp-binding.png)

O comando `show ip dhcp binding` confirma concessões do tipo **Automatic** — inclusive `.65` e `.66` na VLAN 20, provando que a entrega é dinâmica (não estática).

### Tabela de roteamento

![show ip route](img/show-ip-route.png)

Sub-redes diretamente conectadas + rotas estáticas (`S`) para as redes do outro roteador, via o link `/30`.

### Conectividade fim-a-fim

![Pings fim-a-fim](img/pings.png)

Pings bem-sucedidos entre VLANs de lados opostos do link. O **TTL 126** nos pings para T.I. (`.107`) e Servidor (`.117`) confirma que o pacote atravessou os **dois roteadores**.

---

## 6. Troubleshooting — o caso da VLAN 20 sem DHCP

Com a rede montada, todos os segmentos obtinham endereço via DHCP — **exceto a VLAN 20**. Diagnóstico por **eliminação de camadas**, de cima para baixo:

1. **Camada 3 (roteador):** a subinterface `g0/1.20` estava correta (`encapsulation dot1Q 20` + IP na faixa certa) e o pool `ADMINISTRACAO` apontava para a rede certa. Roteador descartado.
2. **Camada 2 (switch):** `show vlan brief` confirmou a VLAN 20 ativa com as portas certas; `show interfaces trunk` mostrou a VLAN 20 permitida e em *forwarding* no trunk. Switch descartado.
3. **Última camada (host):** o problema estava nos **PCs** — não estavam renovando o pedido de DHCP após a subinterface entrar no ar. Alternando a placa para *Static* e de volta para *DHCP*, o pedido foi refeito e o endereço concedido.

**Resultado:** toda a rede passou a receber endereços dinamicamente e a conectividade fim-a-fim foi validada.

---

## 7. Lições Aprendidas

- **O pool DHCP só responde por uma interface com IP na mesma rede do pool.** Configurar o pool antes da subinterface subir faz o cliente cair em APIPA — e ele não re-solicita sozinho.
- **Excluir sempre o gateway do range do pool** (`ip dhcp excluded-address`), ou o DHCP acaba entregando o IP do próprio roteador e gera conflito.
- **Precisão de máscara em rota estática é crítica.** A máscara de uma rota deve casar exatamente com o CIDR da sub-rede de destino — um `/26` no lugar de um `/27` pode "engolir" sub-redes vizinhas e quebrar o roteamento. No papel é detalhe; no IOS, o roteador obedece exatamente o que foi digitado.
- **Troubleshooting é método, não sorte.** Eliminar suspeitos camada por camada isola o problema mais rápido do que sair mexendo em tudo.

---

<sub>Executado em ambiente virtual autorizado (Cisco Packet Tracer). Módulo 1 da trilha DEV + Security.</sub>
