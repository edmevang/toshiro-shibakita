# Desafio prático
Docker: Utilização prática no Cenário de Microsserviços no bootcamp Linux Experience

## Contextualização
Este projeto desafiador visa a implementação de uma estrutura de **Microsserviços** com as melhores práticas do mercado internacional e as benesses que esta proporciona com o desmembramento em *serviços independentes*.

Estes serviços (antes com *arquitetura monolítica*) agora separados comunicam-se entre si usando **APIs** proporcionando agilidade entre os times de desenvolvimento, o uso de linguagens de programação diversas entre os serviços, a **escalabilidade**  é aplicada de acordo com a necessidade para cada serviço, etc.

É aplicado o conceito de **clusterização** onde pressupõe que você tenha três máquinas virtuais rodando na AWS e com as portas padrão liberadas dos serviços utilizados neste lab.
Trabalharemos com o banco de dados **mysql**, o **servidor web apache** + **PHP** e o proxy **NGINX**.

O orquestrador dos containers usado é o **Docker Swarm** ao grupo de hosts do docker formando um cluster Swarm.

Para entendimento as três VM´s na **AWS** rodarão o **Ubuntu Linux** e terão o nome como *aws-1*, *aws-2* e *aws-3* sendo que a aws-1 será a LEADER e as demais WORKER no Docker Swarm.

## Passos iniciais
1. Pegar o IP Público da VM aws-1 para acesso remoto no terminal. 
    No *Windows* utilize o *Putty* e no Linux o seu próprio *terminal* como *root*.
2. Instalar na VM aws-1 o *docker* com as instruções em <https://docs.docker.com/engine/install/ubuntu/>.
3. Criar na VM aws-1 os **volumes** do app e do mysql com `docker volume create app` e `docker volume create data` e o container *docker* mysql com `docker run -e MYSQL_ROOT_PASSWORD=Senha123 -e MYSQL_DATABASE=meubanco --name container-mysql -d -p 3306:3306 --mount type=volume,src=data,dst=/var/lib/mysql/ mysql:5.7`
4. Acessar o *container docker mysql* com `docker exec -it container-mysql bash`. 
   Use o comando `docker start container-mysql` para ativá-lo quando for necessário caso reinice a sua máquina, por exemplo.
5. Logar-se no *mysql* com o comando `mysql -uroot -pSenha123;`.
6. Setar o banco de dados para uso com o comando `use meubanco;`.
7. Criar a tabela dados conforme o script *banco.sql*.
8. Confirmar se a tabela foi criada com o clássico `select * from dados;` e saia do mysql.
9.  Criar a *aplicação PHP* em */var/lib/docker/volumes/app/_data* e fazendo as adaptações no código PHP `index.php` e informe nele o IP Público na VM aws-1 em `$servername` e a senha `$password` para acesso ao banco definida no **passo 3** na criação do container mysql.
10. Criar um container do PHP7 nesta máquina virtual e diretório com o servidor Web Apache com `docker run --name web-server -dt -p 80:80 --mount type=volume,src=app,dst=/app/ webdevops/php-apache:alpine-php7`.
11. Checar com `docker ps` e os containers do apache com php7 e mysql estão rodando.
12. Abrir o navegador e inserir o IP Público da VM aws-1; isto executará a página `index.php` e fará a inserção de registros aletórios no banco de dados mysql *meubanco* e averigue na sua IDE de preferência a inclusão dos registros.
### Estressando o Container ###
Pergunta: Vou precisar aumentar a quantidade de máquinas, as três VM´s que eu tenho na AWS EC2 ???
Os passos abaixo são para responder a pergunta acima.

1. Acessar o site **loader.io** e criar uma conta gratuíta nele.
2. Já logado neste serviço, informar o IP Público da VM aws-1 em *Domain* após clicar em *Target Host* e *New target host*; Copiar o nome do arquivo informado pelo serviço *loader.io* parecido como *loaderio-hash1* no volume espelhado dentro do container com o mesmo nome.
    Os comandos a seguir são o que devem ser executados no container da VM aws-1 e alterado o nome do arquivo informado pelo serviço *loader.io*:
    `cd /var/lib/dpcler/volumes/app/_data/`
    `sudo nano loaderio-hash-informada-no-loaderio.txt` e colocar o nome deste arquivo como conteúdo deste arquivo, desprezando a extensão txt.
3. No serviço *loader.io* clique no botao **Verify**. Clique em *Tests*, *Add new test* e siga a sugestão abaixo para preenchimento:
    *Name: Teste Docker*
    *Test type: Clients per test*
    *Clients: 250*
    *Duration: 1 min*
    *Method: GET*
    *Protocol: HTTP*
    *Host: informe o IP público da VM aws-1*
    *Path: informe o app `index.php` que fará a inclusão dos registros no mysql.
    Clique no botão **Run Test**
4. Conecte-se no banco de dados do container mysql com qualquer IDE e verifique se as inclusões dos testes estão sendo feitas nele!
### Iniciando um cluster Swarm ###
O container *web-server* na VM aws-1 está isolado e será colocado em vários servidores.
1. Por isso vamos destruí-lo com o comando `docker rm --force web-server` na VM aws-1.
2. O comando `docker swarm init` cria um cluster Swarm e com isso será possível replicar o container web-server em vários servidores. A porta 2377 deverá estar liberada previamente na AWS nas VM´s.
    Esse comando cria uma saída de comando que deverá ser executado nas outras VM´s aws-2 e aws-3 que farão parte do cluster Swarm como *workers*; a VM aws-1 é a *leader*.
    A saída de comando tem este formato e você deverá copiá-la após a execução do comando `docker swarm init`: `docker swarm join --token hash-informada ip-informado:2377`

### Criando um serviço no cluster ###
1. Na VM aws-1 verifique com o comando `docker node ls` os nós pertencentes ao cluster recém-criado.
Se tudo ocorreu bem você terá o hostname aws-1 como *Leader* e como *Worker* as máquinas aws-2 e aws-3 e prontas para receber containers.
2. Agora vamos criar um serviço de containers *web server* na VM aws-1 que serão replicados dentro deste cluster nestas VM´s com a instrução `docker service create --name web-server --replicas 3 -dt -p 80:80 --mount type=volume,src=app,dst=/app/ webdevops/php-apache:alpine-php7`
3. Com o comando `docker service ps web-server` você verá como ficou esta replicação do serviço web-server entre os nós do clusters.

### Replicando um volume dentro do cluster ###
É necessário replicar os volumes do serviço para as demais VM´s que não é feito automaticamente!
1. Será necessário instalar o *servidor* NFS na VM aws-1 com o comando `sudo apt-get install nfs-server -y` onde está a aplicação app ou seja o microsserviço.
2. Nas demais VM´s aws-2 e aw-3 deverão ser instalados o NFS como *cliente* com o comando `sudo apt-get install nfs-common -y`.
3. Agora basta informar a pasta a ser replicada onde está o meu app na VM aws-1 no arquivo `/etc/exports`.
    Insira o conteúdo abaixo no arquivo em `/etc/exports` e salve:
    `/var/lib/docker/volumes/app/_data *(rw,sync,subtree_check)`
4. Execute o comando `exportfs -ar` para a validação do passo anterior.
5. Podemos ver o que foi compartilhado nesta VM com o comando `showmount -e` que deverá ser a pasta informada no arquivo `/etc/exports`.
6. Agora vamos montar esta pasta nas VM´s aws-2 e aws-3 com o comando `mount -o v3 ip-do-host-do-container:/var/lib/docker/volumes/app/_data /var/lib/docker/volumes/app/_data`.
7. Verifique se a replicação do conteúdo da VM aws-1 aconteceu nas VM´s aws-2 e aws-3 com o comando `ls` em */var/lib/docker/volumes/app/_data*. Deverão estar replicados o arquivo `index.php` e o arquivo de testes do serviço *loader.io*.

### Criando um proxy com o NGINX ###
Com o *proxy NGINX* as requisições que caiam na VM aws-1 poderão ser replicadas aos demais containers automaticamente dispostos nas VM´s aws-2 e aws-3.
1. Vamos criar um arquivo `nginx.conf` de configuração do proxy em */proxy* com `mkdir /proxy` para informar quais máquinas são pertencentes a ele que são as VM´s aws-1, aws-2, aws-3:
    Utilize o modelo deste arquivo `nginx.conf` e adapte os IP´s em server correspondentes aos seus containers que serão inseridos neste proxy nas VM´s aws-1, aws-2 e aws-3.
    A *porta de acesso ao proxy é a 4500*.
    O cluster separará as requisições automaticamente em cada container da aplicação app.
2. Com o arquivo `nginx.conf` alterado, é necessário mandá-lo para dentro do container do proxy a ser criado e para isso crie o arquivo Dockerfile em */proxy* e use o modelo disponível neste projeto.
   
    No diretório *`/proxy`* você deve ter dois arquivos: `Dockerfile` e `nginx.conf`
1. Agora vamos subir o container com esta configuração criada com o comando `docker build -t proxy-app .` com o nome de *`proxy-app`*.
2. Na VM aws-1 as imagens esperadas do docker quando executado `docker image ls` são:
`proxy-app
mysql
nginx
webdevops/php-apache`
5. Agora vamos subir o container `my-prox-app` com a imagem recém-criada `proxy-app` com o comando `docker container run --name my-proxy-app -dti -p 4500:4500 proxy-app` e estará apenas na VM aws-1.
6. Listando os containers na VM aws-1 com `docker container ls` o container com a imagem `proxy-app` deve estar ON.

## Estressando o cluster ###
Agora vamos testar se o proxy vai distribuir a carga para todo o cluster com o auxílio do serviço *loader.io*.
1.) No serviço *loader.io* a porta do host da VM aws-1 será a 4500 que é a do proxy NGINX. Isso gerará um novo arquivo de identificação que deverá ser copiado o seu nome e extensão e replicado como um novo arquivo e o seu conteúdo é o seu nome, desprezando a sua extensão. 

    Faça isso na VM aws-1 no caminho:

    `/var/lib/docker/volumes/app/_data`

    Neste diretório você deverá ter um arquivo como:
    `index.php
    loaderio-hash1.txt
    loaderio-hash2.txt` >>>>>> Este aqui é o novo arquivo que você criou neste passo e o 
                                                        seu conteúdo é o seu próprio nome do arquivo !!!

2.) Vá para o serviço do *loader.io* e clique no botão **Verify** e aguarde os resultados.
3.) Vamos executar os testes de estresses no cluster:
    Clique em *Tests*
    *Add New Test*
    *GET*
    *HTTP*
    Preencha os demais campos como desejado e observe se o host da VM aws-1 está apontado para a porta do proxy que é a 4500.
    `index.php` como Path.

4.) Verifique na sua IDE do mysql ser os hosts estão sendo alterados, ou seja, se a inclusão dos registros estão sendo feitas de hosts diferentes.

















Para instalar o mysql nesta VM aws-1 basta no console como usuário root digitar: apt-get install mysql-server -y

Muito se tem falado de containers e consequentemente do Docker no ambiente de desenvolvimento. Mas qual a real função de um container no cenários de microsserviços? Qual a real função e quais exemplos práticos podem ser aplicados no dia a dia? Essas são algumas das questões que serão abordadas de forma prática pelo Expert Instructor Denilson Bonatti nesta Live Coding. IMPORTANTE: Agora nossas Live Codings acontecerão no canal oficial da dio._ no YouTube. Então, já corre lá e ative o lembrete! Pré-requisitos: Conhecimentos básicos em Linux, Docker e AWS.

