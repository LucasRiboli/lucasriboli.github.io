---
author: "Lucas"
title: "Operadores Além do K8s: IaC Reativa com Primitivos do Linux"
date: "2025-09-02"
---

# Operadores Além do K8s: IaC Reativa com Primitivos do Linux

Volta e meia eu fico muito aficionado por algum tema da computação. Às vezes é um assunto já batido, que encontro de monte por aí, mas geralmente é algo mais específico, que alguns conhecem e poucos usam.  
O tema da vez foi o **Incus**.

Quando me convidaram para falar em uma apresentação, fiquei empolgado por dois motivos: primeiro, porque seria a minha primeira apresentação, segundo, porque eu poderia falar sobre Incus e LXD.  
Comecei a pensar em como abordar e percebi que poderia acabar trazendo apenas vários *hands-on* e conceitos de virtualização. Fiquei tentado a fazer uma linha quase histórica sobre virtualização e seus usos, mas me imaginei falando 40 minutos só disso… e talvez eu não fosse a melhor pessoa para esse tipo de narrativa (nem ficaria realmente feliz fazendo).

Então decidi retomar alguns tópicos que vinha pensando e conceitos que vi quando estive no MGC: daemon, self-hosted, o próprio Incus, Golang e… **Operadores**. Nesse momento, resolvi juntar tudo e focar em algo mais prático de laboratório: explicar algo próximo do dia a dia do dev (talvez não de todos), mas ao mesmo tempo experimentar umas coisas.

E assim surgiu o tema: **Operadores Além do K8s: IaC Reativa com Primitivos do Linux**.

Mas para chegar no lab precisamos entender algumas coisas.

---

## Mas afinal, o que são operadores?

Para explicar isso, precisamos falar de quem está sempre atrelado a eles (mas talvez não devesse estar): o **Kubernetes**.

De forma simples, o K8s é um **orquestrador de containers**. Mas o que significa orquestrar? Nesse caso, é automatizar deploy, escalabilidade e gerenciamento de aplicações.

Simplista? Talvez. Então vamos um pouco mais fundo.

O K8s foi feito para ambientes grandes, com muitos componentes — o famoso mundo de **microserviços**. Ele lida muito bem com sistemas distribuídos, e por isso exige uma **arquitetura distribuída** (embora haja controvérsias).

No coração dessa arquitetura, temos duas entidades principais:

- **Control Plane**: como o nome indica, é o plano de controle. Ele orquestra as requisições pelo API Server, guarda estados no ETCD (banco NoSQL do K8s), agenda cada pod e mantém monitoramento constante do estado do sistema, garantindo resiliência.
- **Worker Nodes**: onde vivem os pods. Aqui temos o agente que executa as ações, quem gerencia a rede e o runtime dos containers.

Resumindo: quando você cria seu YAML, define um estado desejado para o cluster. O API Server recebe essa intenção, salva no ETCD e passa a ordem para o Scheduler decidir onde colocar cada recurso. Assim que possui essa informação, o Scheduler instrui o Kubelet a provisionar os pods.

Os Controllers garantem que tudo fique no estado correto. Se um pod falha, o Controller que está monitorando cria outro — desde que isso esteja definido, claro.

Esse design todo tem um núcleo quando falamos em resiliência: o **control loop**. É nele que a mágica acontece. Observar o estado atual, comparar com o desejado e agir se necessário. Isso é exatamente o que os operadores fazem.

![Arquitetura do K8s na minha visão](k8sarq.png)

---

## Operadores: o SRE codificado

Operadores são como pequenos K8s que automatizam conhecimento específico de domínio. É como se um SRE experiente estivesse codificado em software, pronto para agir dentro do cluster.

Exemplo prático:  
o K8s escala pods baseado em CPU e memória. Mas e se quisermos algo mais inteligente, como escalar com base no tamanho de uma fila de mensagens?  

O operador enxerga o API Server através de um **CRD (Custom Resource Definition)**. Ele entende o YAML do desejo do usuário e atua como um controller especializado. Nesse caso, ele observaria a fila, contaria mensagens e, se necessário, aumentaria ou reduziria o número de pods.

Percebe o padrão? O operador replica o que um SRE faria manualmente: observar métricas específicas do domínio, tomar decisões inteligentes e agir automaticamente.

![Arquitetura dos operadores na minha visão](k8sopearq.png)

---

## Mas… eu já vi isso antes

Quando vi o K8s chamar isso de “Operator Pattern”, fiquei pensando em como eles foram geniais,fiquei impressionado, eles conseguiram dar nome e estrutura a algo que fazíamos há décadas de forma ad-hoc. Tipo eu pensei: *eu já vi esse padrão antes*.  
E vocês também. Só que não necessariamente dentro do K8s.

Primeiro, entendendo no macro ou micro, não sei muito bem:

![Paralelo do Operator Pattern](images/operadorPattern.png)  
*sim, peguei um slide da apresentação, pois só vi esse paralelo depois de escrever o roteiro e agora estou retocando para o texto* 

Na prática, estamos falando de **reconciliação** — o famoso “observe, compare e aplique”. Esse conceito existe há muito tempo:

- **1975–1990**: Unix e Linux faziam isso com cron + shell scripts, trazendo um tom de reconciliação rudimentar:
```
observe = hora
compare = agenda
act = run script
```
- **2005–2010**: ferramentas de config management, como Puppet:
```
observe = estado real da máquina
compare = manifesto
act = install
```
- **2010**: surge o **systemd** (para alegria de muitos e tristeza de outros tantos), com um control loop de falha:
```
observe = daemon status
compare = unit file
act = restart service
```
- **2011–2014**: o boom do IaC com Terraform, com control loop similar ao config management.  
- **2014 – atual**: K8s e operadores consolidam esse padrão.

---

## Amarrando tudo

Nesse ponto, já citei todos os interesses que mencionei no início — exceto dois.  
Trouxe também um pouco de contexto e base do padrão **control loop**. Mas e agora?  

Bom, agora meu divertimento começa (espero que o de vocês também).  
A ideia: criar um **lab** para demonstrar um operador usando primitivos do Linux, um pouco de Go, Incus — e entregar um HPA para containers Incus.  

É isso mesmo. Bora codar um pouco.

Aliás, o código que usei nesse lab está disponível aqui:
https://github.com/LucasRiboli/Forge-Operator-Lab

Bom seguindo a ideia de como criar um passo a passo e demonstrativo (sinto que sou pessimo nisso)

Primeiro passo acredito que seria clonar o repo:

``` bash
git clone https://github.com/LucasRiboli/Forge-Operator-Lab.git
cd Forge-Operator-Lab
```

O próximo passo, acredito, seria entender primeiro o que eu fiz. Para isso, tem um desenho:

![lab do Operator Pattern](lab.png)  

OBS: Esse lab não é sobre substituir o Kubernetes (ava), mas sim mostrar como o padrão de operadores pode ser aplicado em qualquer infraestrutura — até com primitivos simples do Linux. A ideia é abrir espaço para experimentos, forks e adaptações.

Ok, explicado isso vamos implementar a ideia e de forma talvez meio feia, mas esse é o core da nossa aplicacao: `func runHPALoop`:

Por que ela é o core? Bom, porque nela está o processo do control loop inteiro::

observe:
```go
for _, container := range hpaPods {
		if container.State.Status == "Running" {
			state, err := getInstanceState(container.Name)
			if err != nil {
				log.Printf("Erro ao capturar estado do pod %s: %v", container.Name, err)
				continue
			}

			if state.Memory.Total > 0 {
				porcentagemPod := (state.Memory.Usage * 100) / state.Memory.Total
				sumMemory += porcentagemPod
				runningPods++
				log.Printf("Pod %s: %d%% memória utilizada", container.Name, porcentagemPod)
			}
		}
	}
```


compare:
```go
    if totalMediaMemory >= int64(hpa.HPA.MemoryThreshold) {
		if runningPods < int64(hpa.HPA.MaxPods) {...}
    }
```
E
```go
if totalMediaMemory < int64(hpa.HPA.MemoryThreshold)/2 {
		if runningPods > int64(hpa.HPA.MinPods) {...}
}
```

act:
```go
return createPod(name)
```

```go
return deletePod(podToDelete)
```


Resumindo:

```
observe   -> medir uso de memória dos pods
compare   -> checar se está acima/abaixo do threshold
act       -> criar ou apagar pods
```

Entendendo esse padrão em código, podemos compreender como iniciar essa aplicação toda. De início, eu recomendo instalar o Incus seguindo o passo a passo da documentação oficial (https://linuxcontainers.org/incus/docs/main/installing/)

Após fazer isso pode seguir gerando um executavel do nosso sistema:

```go
go build -o operador cmd/main.go
```

e em seguida instale nosso daemon

```bash
sudo ./operador/install-daemon.sh
```

após isso crie um container com o nome `pod` dentro do incus e estresse ele para ver a magina acontecer

para isso pode rodar

```bash
sudo ./lab-init.sh
```
 
Basicamente, assim você teria seu HPA baseado em memória para o cluster Incus, com o daemon operador como base de controle e o código (que você pode ajustar) operando tudo.

Por fim era isso, gostei mto de fazer esse estudo e esse lab para a apresentacao e transformar ele em texto e segue sendo bom criar labs e compartilhar.