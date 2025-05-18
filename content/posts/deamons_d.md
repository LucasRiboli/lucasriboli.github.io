---
author: "Lucas"
title: "Deamons - Orquestração invisivel"
date: "2025-05-07"
---

## Daemons: Da Física ao Controle de Sistemas Linux

Em 1871, James Clerk Maxwell, para explicar a segunda lei da termodinâmica, propôs um experimento mental bastante curioso. Ele imaginou um ser finito, capaz de manipular moléculas em um sistema fechado, observando e tomando decisões sobre abrir ou fechar uma porta para permitir a passagem de moléculas rápidas. Mais tarde, Lord Kelvin chamou esse ser de "demônio", não no sentido de uma entidade maléfica, mas referindo-se a uma criatura sobrenatural que trabalha nos bastidores, como o conceito de _daemon_ na mitologia grega — seres invisíveis aos humanos, executando tarefas importantes.

Esta analogia com o conceito de "daemon" é frequentemente citada em discussões sobre o termo em computação, embora a ligação histórica direta seja objeto de debate. No contexto dos computadores, o termo assumiu seu próprio significado técnico específico— mas vamos supor que seja verdade, deixa tudo mais legal —.
Mas como essa ideia se aplica na prática?

A ideia central permanece: agora, não mais como um experimento mental, mas como algo concreto. Nos sistemas operacionais, enfrentamos o desafio de manter um sistema complexo sob constante observação e ação, sem a intervenção direta do usuário — ou seja, um verdadeiro _daemon_ grego.

Para isso, surgiram os processos que rodam em segundo plano, conhecidos como daemons. Eles permanecem em estado de dormência — ele nao fica de fato off, mas vou explicar melhor mais a frente — , sendo executados por algum gatilho, em intervalos regulares (geralmente segundos ou milissegundos), ou ainda processando filas continuamente. Existem de todos os tipos e funções, mas a constante é: eles operam de forma autônoma, sem depender da ação do usuário, mantendo o funcionamento do sistema operacional ou gerenciando funções que, eventualmente, podem ser usadas por processos interativos.

Um exemplo importante de sistema de gerenciamento de daemons — e amplamente adotado em muitas distribuições modernas — é o **systemd**. Ele funciona como um gerenciador central: inicia e controla muitos outros serviços, analisando arquivos `.service` para determinar a ordem de execução, localização do binário do processo, e como gerenciar suas entradas e saídas.

Na prática, daemons são a base do Linux, atuando em diversos níveis, desde funções próximas ao hardware até aplicações de usuário. Quase tudo o que usamos ou vemos em nosso workflow tem um daemon por trás. E o melhor: podemos criar nossos próprios daemons para executar tarefas que não requerem nossa atenção direta, funcionando em segundo plano.

Hoje, em muitos sistemas modernos, o systemd é um gerenciador de daemons amplamente utilizado, embora existam alternativas como OpenRC, runit e s6 que têm seus próprios méritos e defensores. Com o systemd, podemos, por exemplo, criar um serviço que monitora a temperatura do hardware e gera logs automaticamente, ou executar aplicações "one shot" sob seu controle, gerenciando sua execução, status e eventuais erros.

Claro, existem diversos outros daemons — como eu disse, eles estão por tudo e até onde a gente talvez (?) nem imagine — como o Apache HTTP Server (httpd), que é um servidor web popular que opera como daemon. Existe o crond, responsável pelas suas tarefas agendadas presentes nos cronjobs, e esse cara tem um ponto interessante, pois se pensarmos de uma forma direta, ele executa tarefas que criamos e pedimos para rodar em determinados momentos ou de tanto em tanto tempo e quando não rodam eles ficam parados, muito semelhante aos próprios daemons!

Seriam meus schedulers daemons também? A linha pode ser tênue, pois compartilham características. A principal diferença é que muitos daemons não estão realmente "dormentes" como se poderia pensar - estão ativamente executando, monitorando eventos, interfaces ou portas, estão em Interruptible Sleep (S) ou Uninterruptible Sleep(D), sendo esperando algum evento e aguardando uma operação I/O ser concluída respectivamente. Alguns utilizam polling contínuo, outros operam baseados em eventos do sistema ou filas de tarefas. Na verdade, tanto daemons quanto schedulers podem enfrentar atrasos devido a condições do sistema, embora daemons sejam normalmente projetados para maximizar a eficiência e resposta.

É interessante também saber que existem várias formas de criar um daemon, algumas antigas e usuais, outras modernas e muito resilientes, ou meio-termo.

Uma forma tradicional seria um script no init.d, que faz parte do sistema de inicialização SysV. Este modelo precedeu o systemd e foi o padrão em distribuições Linux por muitos anos. O SysV init utiliza scripts em /etc/init.d/ para iniciar, parar e gerenciar serviços em diferentes runlevels, oferecendo um sistema mais simples mas menos paralelizado que o systemd. Essa abordagem ainda é suportada em muitas distribuições por razões de compatibilidade.

Também podemos criar scripts diretos com loop ou observando filas sem nenhum controle, possivelmente a forma mais fácil mas a mais problemática: a aplicação sem nenhum controle, gerenciamento etc.

Mas nem tudo é tão preto no branco assim, dentro da comunida Linux, há quem veja o systemd como um quebra de paradigmas inaceitavel. A filosofia Unix prega que "faça uma coisa e faça bem feito" ("Do One Thing and Do It Well"). O systemd faz muitas coisas, para alguns isso é praticidade e um design de aplicação muito favoravel para outros é complexidade e risco de mais. Ouve alguns possicionamentos de algumas figurinhas famosas na comunidade como o de Linux Torvalds que ve de forma neutro e até positiva e Devuan que sugeriu como fork do debian evitar o uso de systemd.

## Hands On - Systemd

Mas não basta entender o conceito — como isso aparece no Linux real que usamos todos os dias? Vamos olhar um pouco para a prática.

Fica aqui o alerta, que vou tratar apenas de systemd nesse hands On, nada contra as demais formas de criar um deamon, é apenas pela praticidade e visualização que teremos.

Antes de colocar a mão na massa e criar algum deamon, vamos analisar alguns ja existentes dentro de um pequeno lab, entao bora lá.

### Lab:

### Laboratório de Troubleshooting com Systemd e Journal

A ideia nesse lab é a gente exercitar como seria criar um deamons usando o systemd e fazer troubleshooting usando journalctl e o proprio systemd

#### Parte 1: Criando um Serviço de Teste

#### 1. Crie um script simples que servirá como nosso serviço

```bash
sudo nano /usr/local/bin/meu-servico.sh
```

Adicione o seguinte conteúdo:

```bash
#!/bin/bash
echo "Iniciando meu-servico"
logger "meu-servico iniciado com sucesso"
count=1
while true; do
    echo "meu-servico está funcionando: iteração $count"
    logger "meu-servico está rodando: verificação #$count"
    count=$((count+1))
    sleep 10
done
```
OBS: pode ser qualquer conteudo mas caso queira algo pronto pode usar esse.

#### 2. Torne o script executável

```bash
sudo chmod +x /usr/local/bin/meu-servico.sh
```

#### 3. Crie um arquivo de unidade systemd para o serviço

```bash
sudo nano /etc/systemd/system/meu-servico.service
```

Adicione o seguinte conteúdo:

```ini
[Unit]
Description=Meu servico executando
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/meu-servico.sh
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

SyslogIdentifier=meu-servico

[Install]
WantedBy=multi-user.target
```

#### 4. Recarregue a configuração do systemd, habilite e inicie o serviço

```bash
sudo systemctl daemon-reload
sudo systemctl enable meu-servico.service
sudo systemctl start meu-servico.service
```

#### 5. Verifique se o serviço está funcionando corretamente

```bash
sudo systemctl status meu-servico.service
```

Observe o status (deve estar "active (running)") e as linhas de log recentes.

#### 6. Verifique os logs do serviço

```bash
sudo journalctl -u meu-servico.service -f
```

O parâmetro `-f` (follow) mostra as mensagens em tempo real.

#### Parte 2: Quebrando o Serviço Intencionalmente

Agora vamos quebrar o serviço de várias maneiras para praticar diagnóstico.

#### Cenário 1: Corromper o script do serviço

```bash
sudo nano /usr/local/bin/meu-servico.sh
```

Modifique o script inserindo um erro de sintaxe ou um comando inválido:


Reinicie o serviço:

```bash
sudo systemctl restart meu-servico.service
```

##### Cenário 2: Modificar as permissões do script

```bash
sudo chmod -x /usr/local/bin/meu-servico.sh
sudo systemctl restart meu-servico.service
```

##### Cenário 3: Dependência quebrada

Modifique o arquivo de unidade do serviço para criar uma dependência inexistente:

```bash
sudo nano /etc/systemd/system/meu-servico.service
```

Adicione uma linha de dependência para um serviço inexistente:

```ini
[Unit]
Description=Meu Serviço de Exemplo
After=network.target
Requires=servico-inexistente.service
```

Recarregue e tente reiniciar:

```bash
sudo systemctl daemon-reload
sudo systemctl restart meu-servico.service
```

##### Parte 3: Investigação e Diagnóstico

Agora vamos usar as ferramentas do systemd e journal para diagnosticar os problemas.

##### 1. Verificar status do serviço

```bash
sudo systemctl status meu-servico.service
```

Observe os códigos de erro, mensagens e status.

##### 2. Analisar logs do journal

```bash
# Ver todos os logs do serviço
sudo journalctl -u meu-servico.service

# Ver apenas os logs de erro
sudo journalctl -u meu-servico.service -p err

# Ver logs de um período específico
sudo journalctl -u meu-servico.service --since "10 minutes ago"

# Ver logs do boot atual
sudo journalctl -u meu-servico.service -b
```

##### 3. Verificar dependências do serviço

```bash
sudo systemctl list-dependencies meu-servico.service
```

##### 4. Verificar configuração do serviço

```bash
sudo systemctl cat meu-servico.service
```

##### 5. Verificar processos

```bash
ps aux | grep meu-servico
```

##### Parte 4: Corrigindo os Problemas

Agora vamos corrigir os problemas que criamos:

##### Corrigir Cenário 1 (erro de sintaxe)

```bash
sudo nano /usr/local/bin/meu-servico.sh
```

DEsfaca o erro de sintexe ou comando invalido

##### Corrigir Cenário 2 (permissões)

```bash
sudo chmod +x /usr/local/bin/meu-servico.sh
```

##### Corrigir Cenário 3 (dependência quebrada)

```bash
sudo nano /etc/systemd/system/meu-servico.service
```

Remova a linha `Requires=servico-inexistente.service`.

##### Recarregar e reiniciar

```bash
sudo systemctl daemon-reload
sudo systemctl restart meu-servico.service
```

## Conclusão

Acredito que passamos por diversos pontos desde a história e alguns fundamentos dos deamons até o lab com um pouco de mão na massa. Por fim era isso; 
hehehehehe, thank you. - Resident Evil 4, Merchant