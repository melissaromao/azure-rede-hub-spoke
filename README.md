# Azure Rede Hub-Spoke
Implementação de um ambiente de rede de gerenciamento utilizando o modelo **hub-spoke** no **Microsoft Azure**.

> [!NOTE]
> **Hub-Spoke** é uma topologia centralizada em que uma única rede central (hub) se conecta a várias redes periféricas (spokes), onde o hub atua como ponto focal para o tráfego de inter-rede, firewalls e serviços compartilhados.

## Topologia
```mermaid
graph TD
    vmhub["vmhub\n10.0.0.0/16"]

    vmspoke1["vmspoke1\n192.168.0.0/16"]
    vmspoke2["vmspoke2\n172.16.0.0/16"]

    vmspoke1 -- "vmspoke1-to-hub" --> vmhub
    vmhub -- "hub-to-spoke1" --> vmspoke1

    vmspoke2 -- "vmspoke2-to-hub" --> vmhub
    vmhub -- "hub-to-spoke2" --> vmspoke2

    rt1["rt-spoke1\nroute: 172.16.0.0/16 \nsubnet: 192.168.0.0/24"]
    rt2["rt-spoke2\nroute: 192.168.0.0/16 \nsubnet: 172.16.0.0/24"]

    vmspoke1 --- rt1
    vmspoke2 --- rt2
```

## Recursos Criados
| Recurso | Nome | Endereço |
|---|---|---|
| VNET Hub | vmhub | 10.0.0.0/16 |
| VNET Spoke1 | vmspoke1 | 192.168.0.0/16 |
| VNET Spoke2 | vmspoke2 | 172.16.0.0/16 |
| VM Hub | vmhub | 10.0.0.4 |
| VM Spoke1 | vmspoke1 | 192.168.0.4 |
| VM Spoke2 | vmspoke2 | 172.16.0.4 |

---

## Etapas
### 1. Criação das VNETs
Criação das três redes virtuais no **resource-group** `rg-hubspoke`.

![vnets](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/01-vnets.png?raw=true)

### 2. Peerigns
Configuração do **peering** entre **Hub ↔ Spoke1 e Hub ↔ Spoke2**, com status _Fully Synchronized / Connected_.

![spoke1-peering](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/02-peerings-spoke1.png?raw=true)
![spoke2-peering](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/02-peerings-spoke2.png?raw=true)

### 3. Virtual Machines
Criação de três VMs Linux, uma em cada VNET.
![vms](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/03-vms.png?raw=true)

### 4. Route Tables
Criação das **routes tables** `rt-spoke1` e `rt-spoke2`, associados às subnets de cada Spoke.
![routes-tables](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/04-route-tables.png?raw=true)

**Spoke 1: Rotas e Subnet**
Rota para `spoke2` (172.16.0.0/16) via **VirtualAppliance** – 10.0.0.4 (Hub).
Subnet `default` (192.168.0.0/24) associado à VNET `vmspoke1`.
![rt-spoke1](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/04-routes-spoke1.png?raw=true)
![subnet-spoke1](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/04-subnets-spoke1.png?raw=true)

**Spoke 2: Rotas e Subnet**
Rota para `spoke1` (192.168.0.0/16) via **VirtualAppliance** – 10.0.0.4 (Hub).
Subnet `default` (172.16.0.0/24) associado à VNET `vmspoke2`.
![rt-spoke2](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/04-routes-spoke2.png?raw=true)
![subnet-spoke2](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/04-subnets-spoke2.png?raw=true)

### 5. Habilitação IP Forwarding
Ativação do **ip forwarding na interface de rede da VM Hub** (`vmhub118`), permitindo que ela **encaminhe pacotes entre os spokes**.
![ip-forwarding](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/05-ip-forwarding.png?raw=true)

### 6. Configuração VM Hub
Comandos executados na VM Hub para **habilitar o roteamento no nível do sistema operacional Linux**.
```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo iptables -A FORWARD -j ACCEPT
sudo iptables -t nat -A POSTROUTING -j MASQUERADE
sudo apt update && sudo apt install iptables-persistent -y
```
![config-vmhub](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/06-config-vmhub.png?raw=true)

### 7. Resultados
**Roteamento Ativo: Ping entre VMs**
Validação da comunicação entre Spoke2 ↔ Spoke 1 e Hub para Spoke1/Spoke2, confirmando **roteamento via hub**.
![spoke1-spoke2](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/07-results-spoke1-ping-spoke2.png?raw=true)
![spoke2-spoke1](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/07-results-spoke2-ping-spoke1.png?raw=true)


![vmhub-spoke1](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/07-results-vmhub-ping-spoke1.png?raw=true)
![vmhub-spoke2](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/07-results-vmhub-ping-spoke2.png?raw=true)

**Effective Routes: VM Hub**
Rotas efetivas da interface de rede do Hub, mostrando as redes dos Spokes aprendidas via VNET Peering.
![effectives-routes](https://github.com/melissaromao/azure-rede-hub-spoke/blob/main/prints/07-results-effective-routes.png?raw=true)

### Conclusão
- [x] Arquitetura Hub-Spoke implementada
- [x] Hub atua como ponto central de roteamento
- [x] Permite comunicação entre os spokes sem peering direto entre eles
- [x] Gerenciamento e escalabilidade de rede facilitada
