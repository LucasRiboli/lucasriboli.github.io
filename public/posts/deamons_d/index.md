# Daemons


## Daemons: Da Física (?) ao Controle de Sistemas Linux

Se tem conceitos e histórias que eu sempre gostei são as relacionadas a fisica, mitologia e computação. E passando por ai na interwebs e dia a dia de trabalho me deparei certa vez com os DAEMONS, bom O nome ja chama a atenção e na minha opinião as relações e conceitos também então, bora ver mais.   

Em 1871 (embora ja houvesse citado ela 1867), James Clerk Maxwell, para explicar a segunda lei da termodinâmica, propôs um experimento mental que refutasse ela. Ele imaginou um ser finito, capaz de manipular moléculas em um sistema fechado, que observava e tomava decisões sobre abrir ou fechar uma porta para permitir a passagem de moléculas rápidas, basicamente de acordo com ele um ser impossivel de existir. Mais tarde, Lord Kelvin chamou esse ser de "demônio", não no sentido ruim da coisa (ele nao quis dizer, mochila de criança, 7 pele, sintéco gelado...), mas referindo-se a uma criatura sobrenatural que trabalha nos bastidores, o conceito de _daemon_ da mitologia grega, seres invisíveis aos humanos, executando tarefas importantes.

Este conceito de "daemon" é frequentemente associado na computação, embora a ligação talvez nem exista, no contexto dos computadores, o termo assumiu seu próprio significado técnico específico (mas vamos supor que existe a ligação, deixa tudo mais legal).

Mas porque teria essa ligação?

Bom a ideia geral é a mesma, mas não mais como um experimento mental, mas como algo virtual. Nos sistemas operacionais, temos que manter o sistema complexo sob constante observação e ação, sem a intervenção direta do usuário, ou seja, um verdadeiro _daemon_ grego.

E voiala temos ai, os Daemons do linux. Eles permanecem em estado de dormência (ele nao fica de fato off, mas vou explicar melhor mais pra frente), sendo executados por algum gatilho, em intervalos regulares (geralmente segundos ou milissegundos), ou ainda processando filas continuamente. Existem de todos os tipos e funções, mas a constante é: eles operam de forma autônoma, sem depender da ação do usuário, mantendo o funcionamento do sistema operacional ou gerenciando funções que, eventualmente, podem ser usadas por processos interativos.

Um exemplo importante de sistema de gerenciamento de daemons e amplamente adotado em muitas distribuições modernas, é o **systemd**. Ele funciona como um gerenciador central: inicia e controla muitos outros serviços, analisando arquivos `.service` para determinar a ordem de execução, localização do binário do processo, e como gerenciar suas entradas e saídas.

Na prática, daemons são a base do Linux, atuando em diversos níveis, desde funções próximas ao hardware até aplicações de usuário. Quase tudo o que usamos ou vemos em nosso workflow tem um daemon por trás. E o melhor, podemos criar nossos próprios daemons para executar tarefas que não requerem nossa atenção direta, funcionando em segundo plano.

Hoje, em muitos sistemas modernos, o systemd é um gerenciador de daemons amplamente utilizado, embora existam alternativas como OpenRC, runit e s6 que têm seus próprios méritos e defensores. Com o systemd, podemos, por exemplo, criar um serviço que monitora a temperatura do hardware e gera logs automaticamente, ou executar aplicações "one shot" sob seu controle, gerenciando sua execução, status e eventuais erros.

Claro, existem diversos outros daemons, como eu disse, eles estão por tudo e até onde a gente talvez (?) nem imagine, como o Apache HTTP Server (httpd), que é um servidor web popular que opera como daemon (eu não sabia até ir atrás). Existe o crond, responsável pelas suas tarefas agendadas presentes nos cronjobs, e esse cara tem um ponto interessante, pois se pensarmos de uma forma direta, ele executa tarefas que criamos e pedimos para rodar em determinados momentos ou de tanto em tanto tempo e quando não rodam eles ficam parados, muito semelhante aos próprios daemons!

Seria então meus schedulers daemons também? A linha pode ser tênue, pois compartilham características. A principal diferença é que muitos daemons não estão realmente "dormentes" eles estao na verdade assumindo diferentes "estados". (eu disse que falaria dnv) 

**Interruptible Sleep (S)**: Nesse estado eles estão ativamente executando, monitorando eventos, interfaces ou portas por exemplo tem o sshd que esta sempre esperando conexoes na porta 22 ou o cupsd que esta aguardando alguém o enviar algo para impressão.

**Uninterruptible Sleep(D)**: Ja nesse estado esta aguardando uma operação I/O ser concluída, por exemplo temos o rsync que durante a cópia de arquivos grandes, pode aparecer em estado D.

Ou seja, alguns utilizam polling, outros operam baseados em eventos do sistema ou filas de tarefas, etc. Na verdade, tanto daemons quanto schedulers podem enfrentar atrasos devido a condições do sistema, embora daemons sejam normalmente projetados para maximizar a eficiência e resposta.

É interessante também saber que existem várias formas de criar um daemon, algumas antigas e usuais, outras modernas e muito resilientes, ou meio-termo.

Uma forma tradicional seria um script no init.d, que faz parte do sistema de inicialização SysV. Este modelo precedeu o systemd e foi o padrão em distribuições Linux por muitos anos. O SysV init utiliza scripts em /etc/init.d/ para iniciar, parar e gerenciar serviços em diferentes runlevels, oferecendo um sistema mais simples mas menos paralelizado que o systemd. Essa abordagem ainda é suportada em muitas distribuições por razões de compatibilidade.

Também podemos criar scripts diretos com loop ou observando filas sem nenhum controle, possivelmente a forma mais fácil, mas a mais problemática: a aplicação sem nenhum controle, gerenciamento etc.

Mas nem tudo é tão preto no branco assim, dentro da comunida Linux, há quem veja o systemd como um quebra de paradigmas inaceitavel. A filosofia Unix prega que "faça uma coisa e faça bem feito" ("Do One Thing and Do It Well"). O systemd faz muitas coisas, para alguns isso é praticidade e um design de aplicação muito favoravel para outros é complexidade e risco de mais.

## Hands On - Systemd

Bom vimos e embora eu ache muito legal a ideia, vamos com um pouco de pratica agora.

Fica aqui o alerta, que vou tratar apenas de systemd nesse hands On, nada contra as demais formas de criar um deamon, é apenas pela praticidade e visualização que teremos.

Antes de colocar a mão na massa e criar algum deamon, vamos analisar alguns ja existentes dentro de um pequeno lab, entao bora lá.

### Lab:

### Laboratório de Troubleshooting com Systemd e Journal

A ideia nesse lab é a gente exercitar como seria criar um daemons usando o systemd e fazer troubleshooting usando journalctl e o proprio systemd

#### Parte 1:

#### 1.

Crie um script simples que servirá como nosso serviço

```bash
sudo vi /usr/local/bin/meu-servico.sh
```

E dentro do servico vamos adicionar um pequeno loop feito com bash.

```bash
#!/bin/bash
echo "Iniciando meu-servico"
logger "meu-servico iniciado"
count=1
while true; do
    echo "meu-servico está funcionando: iteração $count"
    logger "meu-servico está rodando: verificação #$count"
    count=$((count+1))
    sleep 15
done
```
**OBS**: pode ser qualquer conteudo mas caso queira algo pronto pode usar esse.

#### 2.

Agora vamos permitir que esse servico rode

```bash
sudo chmod +x /usr/local/bin/meu-servico.sh
```

#### 3. 

Beleza, se tudo certo até aqui, agora vamos criar um arquivo de config do systemd para o serviço

```bash
sudo vi /etc/systemd/system/meu-servico.service
```

Adicione o seguinte conteúdo:

```ini
[Unit]
Description=Meu servico de teste
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

#### 4.

Ok, agora vamos iniciar tudo. Pra isso seguimos esse passo a passo:

```bash
sudo systemctl daemon-reload
sudo systemctl enable meu-servico.service
sudo systemctl start meu-servico.service
```

#### 5

Pra verificar se o serviço está funcionando corretamente, podemos executar o seguinte comando:

```bash
sudo systemctl status meu-servico.service
```

Se ver um running, parabéns ta feito o nosso primeiro deamon usando systemd.

#### 6.

Agora vamos verificar os logs do nosso servico

```bash
sudo journalctl -u meu-servico.service -f
```

O parâmetro `-f` (follow) mostra as mensagens em tempo real.

#### Parte 2:

Ah beleza, nosso deamon rodando tudo certo e tudo na paz, mas ta na hora de quebrar tudo...

Vamos quebrar o serviço de várias maneiras para praticar troubleshooting.

#### Cenário 1: Corromper o script do serviço

Bora acessar o codigo do scrit novamente

```bash
sudo vi /usr/local/bin/meu-servico.sh
```

Vamos inserir um erro de sintaxe qualquer:


Reinicie o serviço:

```bash
sudo systemctl restart meu-servico.service
```

##### Cenário 2: Modificar as permissões do script

```bash
sudo chmod -x /usr/local/bin/meu-servico.sh
sudo systemctl restart meu-servico.service
```

##### Parte 3:

Agora vamos usar as ferramentas do systemd e journal para diagnosticar os problemas.

##### 1. Verificar status do serviço

```bash
sudo systemctl status meu-servico.service
```

Observe os códigos de erro, mensagens e status.


```bash
# Ver todos os logs do serviço
sudo journalctl -u meu-servico.service

# Ver apenas os logs de erro
sudo journalctl -u meu-servico.service -p err

# Ver logs de um período específico
sudo journalctl -u meu-servico.service --since "10 minutes ago"

# Ver logs do boot atual
sudo journalctl -u meu-servico.service -b

# Verificar dependências do serviço
sudo systemctl list-dependencies meu-servico.service

# Verificar configuração do serviço
sudo systemctl cat meu-servico.service

# Verificar processos
ps aux | grep meu-servico
```

Todos esses comandos vão te ajudar no troubleshooting. Recomendo rodar eles e ver o que cada um faz, buscando pegar cada nuance. Agora tenta achar os erros dentro dos logs, quebra novamente (ou mais), corrige eles e reinicia o daemon. No fim a ideia é essa...
Praticar para que se um dia tiver que fazer um e atuar nele, vai estar tudo show.
