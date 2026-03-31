# Guia Profissional para Uso do ns-3

**Data:** Março 2026  
**Versão:** 1.0  
**Repositório:** ns-3-dev-git  

---

## Sumário

1. [Introdução ao ns-3](#introdução-ao-ns3)
2. [Arquitetura e Funcionamento](#arquitetura-e-funcionamento)
3. [Padrões de Design Fundamentais](#padrões-de-design-fundamentais)
4. [Estrutura Modular](#estrutura-modular)
5. [Fluxo de Criação de Simulações](#fluxo-de-criação-de-simulações)
6. [Ferramentas de Análise](#ferramentas-de-análise)
7. [Comandos Essenciais](#comandos-essenciais)
8. [Boas Práticas para TCC](#boas-práticas-para-tcc)
9. [Estrutura Recomendada do Projeto](#estrutura-recomendada-do-projeto)
10. [Exemplo Prático Completo](#exemplo-prático-completo)
11. [Troubleshooting e Erros Comuns](#troubleshooting-e-erros-comuns)

---

## 1. Introdução ao ns-3

### O que é ns-3?

**ns-3** (Network Simulator 3) é um simulador de redes discreto orientado a eventos, escrito em C++ com bindings Python. É a ferramenta padrão para pesquisa acadêmica em redes de computadores.

### Características Principais

| Característica | Descrição |
|---|---|
| **Tipo de Simulação** | Evento discreto com relógio virtual |
| **Linguagem** | C++ 11+ (com Python opcional) |
| **Licença** | GPL-2.0-only (código aberto) |
| **Plataformas** | Linux, macOS, Windows |
| **Módulos** | 44 módulos oficiais + contrib |
| **Exemplos** | 109 programas de exemplo |
| **Build System** | CMake 3.13+ com wrapper `ns3` |

### Quando Usar ns-3

✅ **Use para:**
- Pesquisa acadêmica em protocolos de rede
- Simulação de topologias complexas
- Análise de desempenho de protocolos
- Estudos de congestionamento e QoS
- Testagem de novos algoritmos sem hardware

❌ **NÃO use para:**
- Emular código real de rede (use containers)
- Análise de performance em tempo real
- Simulações que precisam de drivers reais

### Diferença: Simulador vs Emulador

```
┌─────────────────────────────────────────────────────────────┐
│ SIMULADOR (ns-3)                                            │
│ - Modela comportamento de redes                             │
│ - Rápido (simula dias em minutos)                          │
│ - Controle total de parâmetros                             │
│ - Reprodutível 100%                                         │
│ - Ideal para pesquisa                                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ EMULADOR (Docker, Mininet)                                  │
│ - Executa código real de protocolos                         │
│ - Mais lento (realista)                                     │
│ - Dependências do sistema operacional                       │
│ - Difícil de reproduzir                                     │
│ - Ideal para testes práticos                                │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Arquitetura e Funcionamento

### 2.1 Motor de Simulação de Eventos Discretos

ns-3 implementa um motor de simulação síncrono baseado em eventos:

```
┌─────────────────────────────────────────────────────────────┐
│                    Simulador ns-3                           │
├─────────────────────────────────────────────────────────────┤
│  Relógio Virtual: 64 bits (nanosegundos a segundos)        │
├─────────────────────────────────────────────────────────────┤
│  Fila de Eventos (Priority Queue)                           │
│  ┌─────────────────────────────────────────┐               │
│  │ Tempo: 0s    │ Tempo: 1s  │ Tempo: 2s │              │
│  │ [E1, E2]     │ [E3]       │ [E4,E5]   │              │
│  └─────────────────────────────────────────┘               │
├─────────────────────────────────────────────────────────────┤
│  Executor: Processa cada evento instantaneamente            │
│  Após E1, E2 → Relógio = 1s → Processa E3 → ...           │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Fluxo de Execução

```cpp
// Pseudocódigo do scheduler
while (fila_eventos não está vazia) {
    evento = fila_eventos.pop_proximo();  // Próximo por tempo
    relogio_virtual = evento.tempo;
    evento.funcao_callback();             // Executa instantaneamente
}
```

### 2.3 Vantagens desta Arquitetura

1. **Velocidade**: Simula dias de tráfego em segundos
2. **Determinismo**: Mesma seed = mesmos resultados
3. **Controle**: Total sobre parâmetros, sem variáveis ocultas
4. **Escalabilidade**: Simular milhões de nós (com otimizações)

---

## 3. Padrões de Design Fundamentais

### 3.1 Object Model + Smart Pointers (`Ptr<>`)

ns-3 usa contagem de referência automática para gerenciamento de memória:

```cpp
// Criação de objeto
Ptr<Node> n0 = CreateObject<Node>();

// Cópia de referência (incrementa contador)
Ptr<Node> n0_ref = n0;

// Ao sair do escopo, decrementa contador
// Quando contador = 0, objeto é destruído automaticamente
// SEM memory leaks, SEM delete manual!
```

**Benefício**: Evita segmentation faults e memory leaks em código complexo.

### 3.2 Attribute System (Configuração Dinâmica)

O sistema de atributos permite configuração sem recompilação:

```cpp
// Nível 1: Padrão (hardcoded)
class Node : public Object {
    uint32_t m_id = 0;  // padrão
};

// Nível 2: Config File
// config.xml
<Node id="5"/>

// Nível 3: Linha de Comando
// ./ns3 run programa --ns3::Node::id=10

// Nível 4: Código
dev->SetAttribute("DataRate", StringValue("10Mbps"));

// Nível 5: Callback (dinâmico durante simulação)
dev->TraceConnect("PhyTxBegin", MakeCallback(&MyCallback));

// Nível 6: Validação (controle de entrada)
// Rejeita valores inválidos antes de usar
```

**Precedência**: Código > CLI > Config File > Padrão

**Exemplo prático:**
```cpp
// Criar dispositivo point-to-point
PointToPointHelper p2p;
p2p.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
p2p.SetChannelAttribute("Delay", StringValue("2ms"));
// Esses atributos são aplicados a TODOS os dispositivos criados
```

### 3.3 Callback System (Desacoplamento)

Callbacks permitem reação a eventos sem acoplamento direto:

```cpp
// Sem callback (acoplamento forte)
class Applicacao {
    Canal* canal;  // Conhece o canal explicitamente
    void EnviarDado() { 
        canal->ProcessarDado();  // Chama diretamente
    }
};

// Com callback (desacoplado)
class Aplicacao {
    Callback<void, Packet*> m_callback;  // Função genérica
    void EnviarDado() {
        m_callback(pacote);  // Chama sem conhecer quem é
    }
};

// Conexão em outro lugar
app->SetCallback(MakeCallback(&Canal::ProcessarDado, canal));
```

**Tipos de callbacks comuns:**

```cpp
// Agendar evento futuro
Simulator::Schedule(Seconds(1.0), &MinhaFuncao);

// Conectar a trace (event hook)
device->TraceConnect("Drop", MakeCallback(&MyClass::OnDrop));

// Registrar callback de retorno
socket->SetRecvCallback(MakeCallback(&App::HandleRead));
```

### 3.4 TypeId (Metadados em Tempo de Execução)

TypeId permite introspection e criação dinâmica de objetos:

```cpp
// Registrar tipo
class MinhaClasse : public Object {
public:
    static TypeId GetTypeId(void) {
        static TypeId tid = TypeId("MinhaClasse")
            .SetParent<Object>()
            .AddConstructor<MinhaClasse>()
            .AddAttribute("Velocidade",
                          "Velocidade em m/s",
                          DoubleValue(10.0),
                          MakeDoubleAccessor(&MinhaClasse::m_velocidade),
                          MakeDoubleChecker<double>());
        return tid;
    }
};

// Uso: consultar tipos em tempo de execução
TypeId tid = Object::GetObject<MinhaClasse>()->GetTypeId();
std::string nome = tid.GetName();  // "MinhaClasse"
```

### 3.5 Helper Classes (Simplificação)

Helpers encapsulam operações comuns:

```cpp
// SEM Helper (verboso e propenso a erros)
Ptr<Node> n0 = CreateObject<Node>();
Ptr<Node> n1 = CreateObject<Node>();
Ptr<PointToPointChannel> channel = CreateObject<PointToPointChannel>();
// ... 20 linhas de configuração ...

// COM Helper (simples e robusto)
NodeContainer nodes;
nodes.Create(2);

PointToPointHelper p2p;
p2p.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
p2p.SetChannelAttribute("Delay", StringValue("2ms"));
NetDeviceContainer devices = p2p.Install(nodes);
```

---

## 4. Estrutura Modular

### 4.1 Hierarquia de Módulos

ns-3 segue uma arquitetura em camadas bem definida:

```
┌────────────────────────────────────────────────────────────┐
│ Camada de Aplicações                                       │
│ (http, voip, bulk-send, on-off, udp-echo, etc.)          │
└────────────────────────────────────────────────────────────┘
                            ↑
┌────────────────────────────────────────────────────────────┐
│ Camada de Transporte                                       │
│ (TCP, UDP, DCCP)                                           │
└────────────────────────────────────────────────────────────┘
                            ↑
┌────────────────────────────────────────────────────────────┐
│ Camada de Rede                                             │
│ (IPv4, IPv6, Routing OLSR/AODV, Queue Management)        │
└────────────────────────────────────────────────────────────┘
                            ↑
┌────────────────────────────────────────────────────────────┐
│ Camada de Enlace                                           │
│ (WiFi, LTE, CSMA, Point-to-Point, Ethernet)              │
└────────────────────────────────────────────────────────────┘
                            ↑
┌────────────────────────────────────────────────────────────┐
│ Camada Core                                                │
│ (Node, Packet, Channel, Timer, Scheduler, Attributes)    │
└────────────────────────────────────────────────────────────┘
```

### 4.2 Módulos Principais (44 no Total)

| Categoria | Módulos | Descrição |
|-----------|---------|-----------|
| **Core** | core, network, common | Base, tipos, algoritmos |
| **Internet** | internet, internet-apps | IPv4/IPv6, TCP, UDP, routing |
| **Enlace** | wifi, lte, csma, p2p | Tecnologias específicas |
| **Roteamento** | olsr, aodv, dsdv, nix-vector | Protocolos de roteamento |
| **Aplicações** | applications | Apps padrão (bulk-send, etc.) |
| **Análise** | flow-monitor, stats | Coleta de dados |
| **Utilitários** | config-store, tap-bridge | Ferramentas auxiliares |

### 4.3 Estrutura Interna de um Módulo

Cada módulo segue um padrão consistente:

```
src/nome-modulo/
├── CMakeLists.txt           # Configuração de build
├── model/
│   ├── classe-principal.h    # Declarações
│   ├── classe-principal.cc   # Implementação
│   └── ...
├── helper/
│   ├── helper-class.h        # API simplificada
│   └── helper-class.cc
├── test/
│   ├── teste-unitario.cc     # Testes (executar com ./test.py)
│   └── ...
├── examples/
│   ├── exemplo-1.cc          # Programa exemplo
│   └── exemplo-2.cc
└── doc/
    ├── modelo.rst            # Documentação Sphinx
    └── figuras/              # Imagens
```

**Exemplo**: Módulo `internet`
- **model/**: Classes IPv4, IPv6, TcpSocket
- **helper/**: InternetStackHelper, Ipv4AddressHelper
- **test/**: testes de protocolo
- **examples/**: tcp-echo-server.cc, udp-echo-server.cc

---

## 5. Fluxo de Criação de Simulações

### 5.1 Estrutura Básica de um Programa ns-3

```cpp
#include "ns3/core-module.h"          // Simulador
#include "ns3/network-module.h"        // Nós, canais
#include "ns3/internet-module.h"       // Protocolos
#include "ns3/point-to-point-module.h" // Link P2P
#include "ns3/applications-module.h"   // Apps

using namespace ns3;

// Logging (opcional)
NS_LOG_COMPONENT_DEFINE("MeuPrograma");

int main(int argc, char *argv[]) {
    // ===== PASSO 1: Configurar logging =====
    LogComponentEnable("UdpEchoServerApplication", LOG_LEVEL_INFO);
    LogComponentEnable("UdpEchoClientApplication", LOG_LEVEL_INFO);

    // ===== PASSO 2: Criar topologia =====
    NodeContainer nodes;
    nodes.Create(2);  // Cria 2 nós
    
    // ===== PASSO 3: Conectar nós com canais =====
    PointToPointHelper pointToPoint;
    pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
    pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));
    NetDeviceContainer devices = pointToPoint.Install(nodes);

    // ===== PASSO 4: Instalar stack de protocolos =====
    InternetStackHelper stack;
    stack.Install(nodes);

    // ===== PASSO 5: Atribuir endereços IP =====
    Ipv4AddressHelper address;
    address.SetBase("10.1.1.0", "255.255.255.0");
    Ipv4InterfaceContainer interfaces = address.Assign(devices);

    // ===== PASSO 6: Criar aplicações =====
    // Servidor
    UdpEchoServerHelper echoServer(9);
    ApplicationContainer serverApps = echoServer.Install(nodes.Get(1));
    serverApps.Start(Seconds(0.0));
    serverApps.Stop(Seconds(10.0));

    // Cliente
    UdpEchoClientHelper echoClient(interfaces.GetAddress(1), 9);
    echoClient.SetAttribute("MaxPackets", UintegerValue(1));
    echoClient.SetAttribute("Interval", TimeValue(Seconds(1.0)));
    echoClient.SetAttribute("PacketSize", UintegerValue(1024));
    ApplicationContainer clientApps = echoClient.Install(nodes.Get(0));
    clientApps.Start(Seconds(1.0));
    clientApps.Stop(Seconds(10.0));

    // ===== PASSO 7: Habilitar tracing (opcional) =====
    pointToPoint.EnablePcapAll("udp-echo");  // Gera arquivos PCAP
    
    // ===== PASSO 8: Executar simulação =====
    Simulator::Stop(Seconds(10.0));
    Simulator::Run();
    Simulator::Destroy();

    return 0;
}
```

### 5.2 Passo a Passo Detalhado

#### **Passo 1: Incluir Headers**
```cpp
#include "ns3/core-module.h"          // Tipos básicos
#include "ns3/network-module.h"        // Node, Channel
#include "ns3/internet-module.h"       // TCP/IP stack
#include "ns3/point-to-point-module.h" // Link P2P
#include "ns3/applications-module.h"   // Aplicações
```

#### **Passo 2: Criar Nós**
```cpp
NodeContainer nodes;
nodes.Create(10);           // Cria 10 nós vazios

Ptr<Node> n0 = nodes.Get(0);  // Acesso individual
```

#### **Passo 3: Criar Canais e Dispositivos**
```cpp
// Point-to-Point
PointToPointHelper p2p;
p2p.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
p2p.SetChannelAttribute("Delay", StringValue("2ms"));
NetDeviceContainer devices01 = p2p.Install(nodes.Get(0), nodes.Get(1));

// WiFi (exemplo alternativo)
WifiHelper wifi;
wifi.SetStandard(WIFI_STANDARD_80211b);
YansWifiPhyHelper wifiPhy;
YansWifiChannelHelper wifiChannel = YansWifiChannelHelper::Default();
wifiPhy.SetChannel(wifiChannel.Create());
// ... mais configuração
```

#### **Passo 4: Instalar Protocolos**
```cpp
InternetStackHelper stack;
stack.Install(nodes);  // Instala IPv4, TCP, UDP em todos os nós
```

#### **Passo 5: Atribuir Endereços IP**
```cpp
Ipv4AddressHelper address;
address.SetBase("10.1.1.0", "255.255.255.0");
Ipv4InterfaceContainer interfaces = address.Assign(devices);

// Ou com roteamento
Ipv4GlobalRoutingHelper::PopulateRoutingTables();
```

#### **Passo 6: Criar Aplicações**
```cpp
// Servidor UDP
UdpEchoServerHelper server(9);
ApplicationContainer serverApps = server.Install(nodes.Get(1));
serverApps.Start(Seconds(0.0));
serverApps.Stop(Seconds(10.0));

// Cliente TCP
BulkSendHelper bulk("ns3::TcpSocketFactory",
                     InetSocketAddress(Ipv4Address("10.1.1.2"), 9));
bulk.SetAttribute("MaxBytes", UintegerValue(1000000));
ApplicationContainer clientApps = bulk.Install(nodes.Get(0));
clientApps.Start(Seconds(0.0));
clientApps.Stop(Seconds(10.0));
```

#### **Passo 7: Habilitar Tracing**
```cpp
// PCAP (compatível com Wireshark)
pointToPoint.EnablePcapAll("meu-programa");
// Gera: meu-programa-0-0.pcap, etc.

// ASCII (eventos em texto)
AsciiTraceHelper ascii;
pointToPoint.EnableAsciiAll(ascii.CreateFileStream("meu-programa.tr"));
```

#### **Passo 8: Executar**
```cpp
Simulator::Stop(Seconds(10.0));   // Tempo máximo
Simulator::Run();                 // Executa eventos
Simulator::Destroy();             // Limpeza
```

---

## 6. Ferramentas de Análise

### 6.1 NS_LOG (Logging Estruturado)

Use `NS_LOG` para debugar sua simulação:

```bash
# Habilitar logs de um componente
NS_LOG="UdpEchoServer" ./ns3 run seu-programa

# Saída:
# [+0.000000000s] [UdpEchoServer]  Servidor escutando em :9
# [+1.000000000s] [UdpEchoServer]  Recebeu pacote de 10.1.1.1:49153
```

**Níveis de log:**
- `LOG_NONE` - Sem saída
- `LOG_ERROR` - Apenas erros
- `LOG_WARN` - Avisos e erros
- `LOG_INFO` - Informações gerais
- `LOG_DEBUG` - Detalhes de debug
- `LOG_LOGIC` - Fluxo lógico
- `LOG_FUNCTION` - Entrada/saída de funções

**Exemplo no código:**
```cpp
NS_LOG_COMPONENT_DEFINE("MeuPrograma");

void MinhaFuncao() {
    NS_LOG_INFO("Iniciando função");
    NS_LOG_DEBUG("Variável x = " << x);
    NS_LOG_ERROR("Erro detectado!");
}
```

**Sintaxe de filtro:**
```bash
# Um componente
NS_LOG="Component" ./ns3 run prog

# Múltiplos componentes
NS_LOG="Component1|Component2" ./ns3 run prog

# Com nível específico
NS_LOG="Component=level_info" ./ns3 run prog

# Com prefixo (tempo, função, etc.)
NS_LOG="Component=level_all|prefix_all" ./ns3 run prog
```

### 6.2 PCAP Tracing (Captura de Pacotes)

Gera arquivos `.pcap` análogos ao Wireshark:

```cpp
PointToPointHelper p2p;
p2p.EnablePcapAll("meu-exp");  // Cria meu-exp-*.pcap

// Analisar com Wireshark
// wireshark meu-exp-0-0.pcap
```

**Estrutura dos arquivos:**
```
meu-exp-0-0.pcap   (link entre nó 0 device 0)
meu-exp-0-1.pcap   (link entre nó 0 device 1)
meu-exp-1-0.pcap   (link entre nó 1 device 0)
```

**Vantagens:**
- Compatível com Wireshark (GUI visual)
- Mostra cada bit transmitido
- Headers completos de protocolos
- Ideal para análise detalhada

### 6.3 Flow Monitor (Estatísticas de Fluxo)

Coleta estatísticas de throughput, latência, taxa de perda:

```cpp
#include "ns3/flow-monitor-helper.h"
#include "ns3/flow-monitor.h"

// Antes de Simulator::Run()
FlowMonitorHelper flowmon;
Ptr<FlowMonitor> monitor = flowmon.InstallAll();

// Depois de Simulator::Run()
monitor->SerializeToXmlFile("flow-monitor.xml", true, true);

// Análise em Python
import xml.etree.ElementTree as ET
tree = ET.parse("flow-monitor.xml")
for flow in tree.iter("Flow"):
    src = flow.get("src")
    dst = flow.get("dst")
    rxBytes = int(flow.find("Rx").get("bytes"))
    txBytes = int(flow.find("Tx").get("bytes"))
    delay = float(flow.find("Delay").get("sum")) / 1e9  # segundos
    throughput = (rxBytes * 8) / delay / 1e6  # Mbps
    print(f"{src} -> {dst}: {throughput:.2f} Mbps, {delay:.3f}s")
```

**Métricas fornecidas:**
- Bytes/Pacotes recebidos e transmitidos
- Taxa de perda
- Atraso (min, max, médio)
- Jitter
- Throughput

### 6.4 ASCII Tracing (Eventos em Texto)

Registra todos os eventos em formato texto:

```cpp
AsciiTraceHelper ascii;
p2p.EnableAsciiAll(ascii.CreateFileStream("meu-exp.tr"));

// Arquivo gerado:
// +0.000000000s 0 0 RX 10.1.1.1 10.1.1.2 IPv4 1234 ALLOW
// +0.000200000s 1 0 RX 10.1.1.2 10.1.1.1 IPv4 40 ALLOW
// +0.000400000s 0 0 DROP 10.1.1.1 10.1.1.2 IPv4 1234 QUEUE
```

**Interpretação:**
- `+0.000000000s` - Tempo de simulação
- `0` - ID do nó
- `0` - ID do dispositivo
- `RX/TX/DROP` - Ação
- `10.1.1.1` - IP origem
- `10.1.1.2` - IP destino
- `IPv4` - Protocolo
- `1234` - Tamanho do pacote
- `ALLOW/QUEUE/DROP` - Resultado

### 6.5 Profiling (Análise de Performance)

Para simulações grandes, optimize performance:

```bash
# Compilar com otimizações
./ns3 configure --build-profile=optimized

# Usar gprof (profiler GNU)
./ns3 run seu-programa --gprof

# Analisar resultados
gprof ./build/src/ns-3-program gmon.out > profile.txt
```

---

## 7. Comandos Essenciais

### 7.1 Configuração e Build

```bash
# Configuração básica
./ns3 configure

# Com exemplos e testes
./ns3 configure --enable-examples --enable-tests

# Mostrar todas as opções
./ns3 configure --help

# Listar alvos disponíveis
./ns3 show targets

# Compilação
./ns3 build

# Compilar alvo específico
./ns3 build core

# Limpeza
./ns3 clean

# Limpeza total (reconfigura)
./ns3 clean && ./ns3 configure --enable-examples --enable-tests
```

### 7.2 Execução

```bash
# Executar programa simples
./ns3 run seu-programa

# Com argumentos
./ns3 run seu-programa --param1=valor1 --param2=valor2

# Com logging
NS_LOG="*=level_all" ./ns3 run seu-programa

# Com debugger (GDB)
./ns3 run seu-programa --command-template "gdb --args %s"

# Com profiler (Valgrind)
./ns3 run seu-programa --command-template "valgrind --leak-check=full %s"
```

### 7.3 Testes

```bash
# Executar todos os testes
./test.py

# Testes de módulo específico
./test.py -s internet

# Testes de exemplo específico
./test.py -e first

# Modo verbose (detalhado)
./test.py -v

# Executar teste com logging
NS_LOG="*" ./test.py -s core
```

### 7.4 Desenvolvimento

```bash
# Criar novo módulo
./utils/create-module.py meu-modulo

# Verificar estilo de código
./utils/check-style.py src/meu-modulo/

# Formatar código
clang-format -i src/meu-modulo/model/*.cc

# Build com debug symbols
./ns3 configure --build-profile=debug
./ns3 build
```

---

## 8. Boas Práticas para TCC

### 8.1 Validação e Veracidade

**Sempre valide seus resultados:**

```cpp
// ✅ BOM: Validar com valores conhecidos
// Simulação com apenas 2 nós, 1 fluxo TCP
// Comparar throughput com fórmula teórica de MSS/(RTT)
double throughput_simulado = (bytes_recebidos * 8) / tempo_total;
double throughput_esperado = (1460 * 8) / (0.004 + 0.001);  // Teórico
NS_ASSERT_MSG(fabs(throughput_simulado - throughput_esperado) < 0.1,
              "Throughput divergiu do esperado!");
```

### 8.2 Reprodutibilidade

**Controle o RNG (Random Number Generator) para reproduzibilidade:**

```cpp
// Seed determinística
RngSeedManager::SetSeed(12345);
RngSeedManager::SetRun(1);

// Mesmo seed = mesmos resultados aleatórios
// Crítico para testes e peer-review em TCC
```

### 8.3 Análise Estatística

**Execute múltiplos runs com análise estatística:**

```bash
#!/bin/bash
# Script para múltiplos runs
for run in {1..30}; do
    NS_RNG_SEED=$run ./ns3 run seu-programa \
        | grep "throughput" >> resultados.txt
done

# Em Python: calcular média e desvio padrão
import numpy as np
dados = np.loadtxt("resultados.txt")
media = np.mean(dados)
std = np.std(dados)
ci_95 = 1.96 * std / np.sqrt(len(dados))
print(f"Throughput: {media:.2f} ± {ci_95:.2f} Mbps (95% CI)")
```

### 8.4 Documentação

**Código bem comentado é essencial:**

```cpp
/**
 * Simula transmissão TCP em link P2P
 * 
 * Topologia:
 *   Nó 0 (Cliente) ---- 5Mbps/2ms ---- Nó 1 (Servidor)
 * 
 * Parâmetros:
 *   - BandWidth: 5 Mbps
 *   - Delay: 2 ms
 *   - Tempo: 10 segundos
 * 
 * Métricas coletadas:
 *   - Throughput (Flow Monitor)
 *   - Latência
 *   - Pacotes perdidos
 */

#include "ns3/core-module.h"
// ... resto do código com comentários linha-por-linha
```

### 8.5 Escalabilidade

**Teste com diferentes escalas:**

```cpp
// Parâmetro ajustável
uint32_t num_nods = std::stoi(argv[1]);  // ./ns3 run prog 100

NodeContainer nodes;
nodes.Create(num_nods);

// Testar: 10, 50, 100, 500 nós
// Medir tempo de simulação e tempo wall-clock
// Gráfico: throughput vs número de nós
```

### 8.6 Versioning

**Use Git para rastrear mudanças:**

```bash
git add seu-programa.cc
git commit -m "Adicionar simulação TCP básica"
git log --oneline seu-programa.cc  # Ver histórico
```

---

## 9. Estrutura Recomendada do Projeto

Para seu projeto de TCC, recomendo a seguinte organização:

```
seu-tcc-ns3/
│
├── README.md                      # Instruções para compilar e executar
├── RESULTADOS.md                  # Documentação dos resultados
│
├── src/
│   └── seu-modulo/               # (Opcional) Módulos customizados
│       ├── model/
│       ├── helper/
│       └── test/
│
├── scratch/                       # Programas de simulação
│   ├── simulacao-basica.cc
│   ├── simulacao-avancada.cc
│   └── comparacao-protocolos.cc
│
├── examples/                      # Exemplos baseados em seu TCC
│   └── seu-cenario.cc
│
├── results/                       # Dados gerados pelas simulações
│   ├── flow-monitor/
│   │   ├── cenario1.xml
│   │   └── cenario2.xml
│   ├── pcap/
│   │   ├── cenario1-*.pcap
│   │   └── cenario2-*.pcap
│   └── logs/
│       ├── cenario1.log
│       └── cenario2.log
│
├── scripts/                       # Ferramentas de análise
│   ├── analise-flow-monitor.py
│   ├── plotar-resultados.py
│   ├── processar-logs.sh
│   └── gerar-graficos.gnuplot
│
├── docs/                          # Documentação do projeto
│   ├── metodologia.md
│   ├── topologias.md
│   ├── parametros.md
│   └── figuras/
│       ├── topologia.png
│       └── fluxo-simulacao.png
│
├── tests/                         # Testes da simulação
│   └── validacao-basica.cc
│
├── Makefile                       # (Opcional) Facilita compilação
├── .gitignore                     # Ignorar arquivos grandes
└── CMakeLists.txt                # (Se criar módulo customizado)
```

**Arquivo `.gitignore` recomendado:**
```
# Build
build/
*.o
*.so

# Resultados
results/
*.xml
*.pcap
*.log
*.txt

# Editor
.vscode/
.idea/
*.swp

# Python
__pycache__/
*.pyc
```

---

## 10. Exemplo Prático Completo

### 10.1 Cenário: Simulação TCP em Topologia Dumbbell

Vou criar um exemplo completo de uma simulação realista:

**Topologia:**
```
Nó 0 (Cliente)
      ↓
  (10Mbps, 10ms)
      ↓
Nó 10 (Router 1) ←→ (5Mbps, 20ms) ←→ Nó 11 (Router 2)
      ↑
  (10Mbps, 10ms)
      ↑
Nó 1 (Cliente)
      ...
Nó 9 (Cliente)
```

**Arquivo: `scratch/tcp-dumbbell.cc`**

```cpp
#include "ns3/core-module.h"
#include "ns3/network-module.h"
#include "ns3/internet-module.h"
#include "ns3/point-to-point-module.h"
#include "ns3/applications-module.h"
#include "ns3/flow-monitor-helper.h"
#include "ns3/ipv4-global-routing-helper.h"

using namespace ns3;

NS_LOG_COMPONENT_DEFINE("TcpDumbbellSimulation");

int main(int argc, char *argv[]) {
    // ===== CONFIGURAÇÃO INICIAL =====
    
    uint32_t nClientes = 10;           // Clientes em cada lado
    std::string dataRate = "10Mbps";    // Taxa de acesso
    std::string delay = "10ms";         // Atraso de acesso
    std::string bottleneck = "5Mbps";   // Gargalo
    std::string bottleneckDelay = "20ms";
    double simTime = 10.0;              // Tempo de simulação
    
    // Parse command-line arguments
    CommandLine cmd;
    cmd.AddValue("nClientes", "Número de clientes em cada lado", nClientes);
    cmd.AddValue("dataRate", "Taxa de dados dos links de acesso", dataRate);
    cmd.AddValue("simTime", "Tempo de simulação", simTime);
    cmd.Parse(argc, argv);
    
    // Habilitar logging
    LogComponentEnable("TcpDumbbellSimulation", LOG_LEVEL_INFO);
    LogComponentEnable("OnOffApplication", LOG_LEVEL_INFO);
    
    // ===== CRIAR TOPOLOGIA =====
    
    NodeContainer serverNodes, clientNodes, routerNodes;
    serverNodes.Create(nClientes);
    clientNodes.Create(nClientes);
    routerNodes.Create(2);
    
    // Nó 0-9: clientes à esquerda
    // Nó 10-19: servidores à direita
    // Nó 20: router esquerda
    // Nó 21: router direita
    
    NS_LOG_INFO("Criados " << nClientes << " clientes e " 
                << nClientes << " servidores");
    
    // ===== CRIAR CANAIS =====
    
    // Links de acesso (Clientes ↔ Router 1)
    PointToPointHelper accessLink;
    accessLink.SetDeviceAttribute("DataRate", StringValue(dataRate));
    accessLink.SetChannelAttribute("Delay", StringValue(delay));
    
    NetDeviceContainer clientDevices, clientRouterDevices;
    for (uint32_t i = 0; i < nClientes; i++) {
        NetDeviceContainer link = accessLink.Install(clientNodes.Get(i),
                                                     routerNodes.Get(0));
        clientDevices.Add(link.Get(0));
        clientRouterDevices.Add(link.Get(1));
    }
    
    // Links de acesso (Servidores ↔ Router 2)
    NetDeviceContainer serverDevices, serverRouterDevices;
    for (uint32_t i = 0; i < nClientes; i++) {
        NetDeviceContainer link = accessLink.Install(serverNodes.Get(i),
                                                     routerNodes.Get(1));
        serverDevices.Add(link.Get(0));
        serverRouterDevices.Add(link.Get(1));
    }
    
    // Link de gargalo (Router 1 ↔ Router 2)
    PointToPointHelper bottleneckLink;
    bottleneckLink.SetDeviceAttribute("DataRate", StringValue(bottleneck));
    bottleneckLink.SetChannelAttribute("Delay", StringValue(bottleneckDelay));
    NetDeviceContainer routerDevices = bottleneckLink.Install(routerNodes);
    
    NS_LOG_INFO("Topologia criada com gargalo de " << bottleneck);
    
    // ===== INSTALAR STACK DE INTERNET =====
    
    InternetStackHelper stack;
    stack.Install(serverNodes);
    stack.Install(clientNodes);
    stack.Install(routerNodes);
    
    // ===== ATRIBUIR ENDEREÇOS IP =====
    
    Ipv4AddressHelper address;
    std::vector<Ipv4InterfaceContainer> clientInterfaces;
    
    // Clientes
    for (uint32_t i = 0; i < nClientes; i++) {
        std::ostringstream subnet;
        subnet << "10." << i << ".0.0";
        address.SetBase(subnet.str().c_str(), "255.255.255.0");
        
        Ipv4InterfaceContainer interfaces = address.Assign(
            NetDeviceContainer(clientDevices.Get(i), 
                              clientRouterDevices.Get(i)));
        clientInterfaces.push_back(interfaces);
    }
    
    // Servidores
    std::vector<Ipv4InterfaceContainer> serverInterfaces;
    for (uint32_t i = 0; i < nClientes; i++) {
        std::ostringstream subnet;
        subnet << "172." << i << ".0.0";
        address.SetBase(subnet.str().c_str(), "255.255.255.0");
        
        Ipv4InterfaceContainer interfaces = address.Assign(
            NetDeviceContainer(serverDevices.Get(i), 
                              serverRouterDevices.Get(i)));
        serverInterfaces.push_back(interfaces);
    }
    
    // Router backbone
    address.SetBase("192.168.1.0", "255.255.255.0");
    address.Assign(routerDevices);
    
    // Roteamento global
    Ipv4GlobalRoutingHelper::PopulateRoutingTables();
    
    NS_LOG_INFO("Endereços IP atribuídos e roteamento configurado");
    
    // ===== CRIAR APLICAÇÕES =====
    
    // Servidores (escutam na porta 9)
    uint16_t sinkPort = 9;
    for (uint32_t i = 0; i < nClientes; i++) {
        PacketSinkHelper sinkHelper("ns3::TcpSocketFactory",
                                    InetSocketAddress(Ipv4Address::GetAny(),
                                                     sinkPort));
        ApplicationContainer sinkApp = sinkHelper.Install(serverNodes.Get(i));
        sinkApp.Start(Seconds(0.0));
        sinkApp.Stop(Seconds(simTime));
    }
    
    // Clientes (enviam dados)
    for (uint32_t i = 0; i < nClientes; i++) {
        BulkSendHelper source("ns3::TcpSocketFactory",
                             InetSocketAddress(serverInterfaces[i].GetAddress(0),
                                              sinkPort));
        source.SetAttribute("MaxBytes", UintegerValue(0));  // Ilimitado
        source.SetAttribute("SendSize", UintegerValue(1460));
        
        ApplicationContainer sourceApp = source.Install(clientNodes.Get(i));
        // Deslocar início de cada cliente para evitar sincronização
        sourceApp.Start(Seconds(i * 0.1));
        sourceApp.Stop(Seconds(simTime));
    }
    
    NS_LOG_INFO("Aplicações TCP criadas: " 
                << nClientes << " clientes e " << nClientes << " servidores");
    
    // ===== HABILITAR TRACING =====
    
    // PCAP
    bottleneckLink.EnablePcapAll("tcp-dumbbell");
    
    // Flow Monitor
    FlowMonitorHelper flowmon;
    Ptr<FlowMonitor> monitor = flowmon.InstallAll();
    
    // ===== EXECUTAR SIMULAÇÃO =====
    
    NS_LOG_INFO("Iniciando simulação de " << simTime << " segundos...");
    
    Simulator::Stop(Seconds(simTime));
    Simulator::Run();
    
    // ===== COLETAR RESULTADOS =====
    
    monitor->SerializeToXmlFile("tcp-dumbbell-flowmon.xml", true, true);
    
    // Imprimir estatísticas
    uint32_t txBytes = 0, rxBytes = 0, txPackets = 0, rxPackets = 0;
    uint32_t lostPackets = 0;
    
    Ptr<Ipv4FlowClassifier> classifier = 
        DynamicCast<Ipv4FlowClassifier>(flowmon.GetClassifier());
    std::map<FlowId, FlowMonitor::FlowStats> stats = monitor->GetFlowStats();
    
    for (std::map<FlowId, FlowMonitor::FlowStats>::const_iterator i 
         = stats.begin(); i != stats.end(); ++i) {
        txBytes += i->second.txBytes;
        rxBytes += i->second.rxBytes;
        txPackets += i->second.txPackets;
        rxPackets += i->second.rxPackets;
        lostPackets += i->second.lostPackets;
    }
    
    double throughput = (rxBytes * 8.0) / (simTime * 1e6);  // Mbps
    double loss_rate = (lostPackets * 100.0) / txPackets;
    
    NS_LOG_INFO("=== RESULTADOS DA SIMULAÇÃO ===");
    NS_LOG_INFO("Throughput total: " << throughput << " Mbps");
    NS_LOG_INFO("Taxa de perda: " << loss_rate << " %");
    NS_LOG_INFO("Pacotes transmitidos: " << txPackets);
    NS_LOG_INFO("Pacotes recebidos: " << rxPackets);
    NS_LOG_INFO("Pacotes perdidos: " << lostPackets);
    
    Simulator::Destroy();
    
    return 0;
}
```

**Para compilar e executar:**

```bash
# Configurar (se necessário)
./ns3 configure --enable-examples

# Compilar
./ns3 build

# Executar
./ns3 run "tcp-dumbbell --nClientes=10"

# Com logging detalhado
NS_LOG="*=level_info|prefix_all" ./ns3 run "tcp-dumbbell --nClientes=10"

# Analisar PCAP com Wireshark
wireshark tcp-dumbbell-0-0.pcap
```

---

## 11. Troubleshooting e Erros Comuns

### 11.1 Erros de Compilação

**Erro: "module not found"**
```
error: ns3/wifi-module.h: No such file or directory
```
**Solução:**
```bash
./ns3 configure --enable-modules=wifi
./ns3 build
```

**Erro: "CMake not found"**
```bash
# Instalar CMake
sudo apt-get install cmake

# Ou em macOS
brew install cmake
```

### 11.2 Problemas de Execução

**Erro: "Segmentation fault"**
- Use Valgrind para detectar memory leaks
```bash
./ns3 run seu-prog --command-template "valgrind --leak-check=full %s"
```

**Erro: "Address already in use"**
- Mudança de porta na aplicação
```cpp
UdpEchoServerHelper server(9);  // Tentar porta diferente
```

### 11.3 Problemas com Resultados

**Problemas: "Resultados não fazem sentido"**

Checklist:
- [ ] Validar topologia com `./ns3 show targets`
- [ ] Verificar que endereços IP estão corretos
- [ ] Confirmar tempo de simulação suficiente
- [ ] Testar com topologia simples primeiro
- [ ] Habilitar `NS_LOG` para debug

**Problema: "Flow Monitor não gera arquivo"**
```cpp
// Versão correta
monitor->SerializeToXmlFile("resultado.xml", true, true);
// Garante flush antes de destruir
```

### 11.4 Performance

**Simulação muito lenta:**

1. Reduzir número de nós
2. Encurtar tempo de simulação
3. Compilar com otimizações
```bash
./ns3 configure --build-profile=optimized
```

4. Usar scheduler otimizado
```cpp
GlobalValue::Bind("SchedulerType", StringValue("ns3::HeapScheduler"));
```

---

## Referências e Recursos

### Documentação Oficial
- **Manual ns-3**: https://www.nsnam.org/docs/manual/html/
- **API Reference**: https://www.nsnam.org/docs/doxygen/
- **Model Library**: https://www.nsnam.org/docs/models/html/

### Exemplos no Repositório
- Localização: `examples/` e `scratch/`
- Total: 109 exemplos prontos para estudar
- Recomendado começar: `first.cc`, `second.cc`, `third.cc`

### Comunidade
- **Mailing list**: https://www.nsnam.org/support/
- **Wiki**: https://www.nsnam.org/wiki/
- **GitHub Issues**: Para reportar bugs

---

## Conclusão

Este guia fornece as informações necessárias para:

✅ Entender a arquitetura de eventos discretos  
✅ Dominar os padrões de design do ns-3  
✅ Criar simulações de rede reprodutíveis  
✅ Coletar e analisar dados de simulação  
✅ Implementar um TCC acadêmico com qualidade  

**Próximos passos:**
1. Execute o exemplo `tcp-dumbbell.cc`
2. Estude os 109 exemplos fornecidos
3. Incremente gradualmente a complexidade
4. Documente suas descobertas
5. Compare com literatura acadêmica

Sucesso em seu TCC! 🚀

---

**Última atualização:** Março 2026  
**Autor:** OpenCode AI Assistant  
**Versão ns-3:** ns-3-dev-git  
