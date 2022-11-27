---
layout: post
title: "Hvordan Jeg Laver REST API'er i AWS"
date: 2022-11-27 23:30:00 +0200
category: "hvordan-jeg"
tags: ["REST API", "AWS", "LAMBDA", "JAVASCRIPT"]
---

## Indledning
I lang tid brugte jeg node.js og express, som jeg hostede i en VPS (Vultr, Hostinger, Digitalocean, etc.), til at lave alle mine REST API'er.
Det var faktisk på mange måder en rigtig god løsning - det var super nemt og billigt at få en server. Jeg brugte f.eks.
$5 pr. måned på en lille server på Vultr.com, som have 1GiB RAM og 2 vCPU-kerner - rigeligt.
Men det endte altid med, at jeg skulle bruge en masse tid på opsætning og vedligeholdelse af serveren. Jeg gad ikke
rigtig at skulle rode med en masse Nginx-konfig-filer hver gang, jeg skulle sætte et nyt API op.

## Cloud
Derfor valgte jeg at kigge på nogle cloud-løsninger. Som jeg skriver dette indlæg, så er der tre modne cloud-udbydere:
* Amazon AWS
* Microsoft Azure
* Google Cloud Platform
Så valget stod mellem disse. Jeg tror egentlig, jeg kunne have valgt hvilken som helst af disse og ville have været glad
for mit valg - så du behøver ikke at gå så meget op i, hvad du vælger.
Jeg endte med at vælge Amazon AWS, fordi det er den ældste, mest modne platform med flest services.

## AWS Services
Når man skal lave et REST API i AWS, så har man nogle forskellige muligheder. Man kan bruge et AWS EC2 instance, som er
din helt egen linux-server, som du får total rådighed over. Men så er vi lidt tilbage til det originale problem, jeg
havde: det tager for lang tid at sætte op. En anden mulighed er at lægge hele sit node.js og express API i en Docker
container og så bruge noget som AWS ECS eller AWS App Runner, som har indbygget load balancer og scaling. Dette er en
super fin løsning, men det kan hurtigt blive lidt dyrt, når man skal have en server online hele tiden. Derfor valgte jeg
at bruge AWS Lambda. AWS Lambda er en serverless løsning - det betyder altså, at API-serveren kun er tændt, når den
behandler requests fra klienter. Det koster altså også kun penge, når API'et faktisk er aktivt; det er dejligt for
18-årige mig, som ikke har særligt mange penge på lommen. Dog er der den ulempe, at serveren så selvfølgelig tager lidt
ekstra tid om at svare, når den er slukket i forvejen. Så hvis serveren ikke har fået et request i et stykke tid, skal
den lave et såkaldt cold boot; det kan godt tage et par sekunder. Hvis serveren ikke er slukket endnu, så er svartiderne
MEGET bedre - lige så gode som en normal server.

## API Gateway
For at gøre hele processen af at lave et REST API med AWS nemmere, så bruger jeg AWS API Gateway. API Gateway er det
yderste lag, hvor alle klienterne laver requests til. API Gateway står for at have styr på alle de forskellige ruter i
dit api, f.eks. "GET /users", "POST /user", "GET /user/:id", etc., og for hvilke AWS services som disse ruter skal integreres med, f.eks.
Lambda.

For hvert nyt REST API, jeg laver, opretter jeg et nyt API Gateway HTTP API og en ny Lambda-funktion.

## Lambda
Når man bruger Lambda, behøver man faktisk ikke at bruge express eller noget lignende til at lave REST API'er; man kan
bare bruge ren javascript. Det synes jeg er en super nem og god løsning. Dog mister man noget funktionalitet, som
express tilbyder.

For hver ny rute, som jeg definerer i API Gateway, laver jeg en integration til min Lambda-funktion. Det er her i lambda
funktionen, at koden til API'et faktisk ligger. Du kan lave en simpel Lambda-funktion med:
```bash
$ mkdir funktion
$ cd funktion/
$ npm init -y
$ touch index.js
```
Den mest simple Lambda-funktion ser således ud:
```javascript
export async function handler(event) {
    return {
	"statusCode": 200,
	"body": "Hej, verden!",
    };
}
```
I alle Lambda-funktioner skal der altid være funktionen handler(event). Denne funktion bliver nemlig kaldt hver gang der
kommer et request fra en klient igennem API Gateway. event-parameteret er vigtigt - det er et javascript-objekt, som
består af en masse info om requestet. Af denne info benytter jeg mig mest af event.requestContext.http.method, som er den
HTTP-metode der bliver brugt (GET, POST, PUT, DELETE, etc.), event.requestContext.http.path, som er den rute der bliver
requestet, f.eks. "/users", event.headers, som er ...ja, de headers der bliver sendt, og til sidst event.body, som er
den body der bliver sendt i f.eks. POST- og PUT-requests.

Lambda forventer at denne funktion, handler(event), returnerer et objekt med en bestemt form. Der skal altid være en
"statusCode" som er en HTTP-statuskode, f.eks. 200, 201, 301, 401, 500, etc. Dem kan du læse mere om [her.](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
Derudover skal der være en "body", som er selve det som API'et returnerer til klienten; det kan f.eks. være en liste over
brugere eller lignende.

Man kan sagtens have en separat Lambda-funktion for hver af sine API-ruter, men det er unødvendigt besværligt. Så
derfor skal vi bruge en måde at kunne skelne mellem de forskellige ruter og HTTP-metoder, som klienten sender.
Jeg gør det således:
```javascript
function buildResponse(statusCode, body) {
    return {
        statusCode: statusCode,
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify(body)
    }
}

export async function handler(event) {
    let method = event.requestContext.http.method;
    let path = event.requestContext.http.path;
    let headers = event.headers;
    let body = JSON.parse(event.body);

    let response;
    switch (true) {
        case method === "GET" && path === "/":
            response = buildResponse(200, event);
            break;
        case method === "POST" && path === "/login":
            response = await postLogin(body);
            break;
	case method == "GET" && path === "/users":
	    response = await getUsers(headers);
	    break;
        case method === "GET" && path === "/session_active":
            response = await getSessionActive(headers);
            break;
        case method === "PUT" && path === "/profile_picture":
            response = await putProfilePicture(body);
            break;
        default:
            response = buildResponse(404, "404 Not Found.");
            break;
    }
    return response;
}
```
Her bruger jeg altså en (ret genial, hvis jeg selv skulle sige det) switch-case til at skelne mellem ruter og HTTP-metoder.
Så skriver jeg selve logikken i andre funktioner.

### Upload til AWS
For at uploade din nye kode til din Lambda-funktion, skal du zippe alle filerne sammen:
```bash
(funktion/) $ zip -r ../funktion.zip .
```
Så kan du gå ind på [console.aws.amazon.com/lambda](https://console.aws.amazon.com/lambda) og gå ind på din
Lambda-funktion og trykke "Upload from" og så ".zip file" og så vælge din nyskabte funktion.zip.

## Konklusion
Jeg har i dette indlæg skrevet om mine problemer med at bruge en VPS til at hoste mine REST API'er, hvilke mulige
cloud-løsninger der er, hvilke AWS services man kan bruge til at hoste REST API'er, hvordan API Gateway virker, hvordan
Lambda virker og til sidst, hvordan man uploader sin Lambda-funktion til AWS.

Jeg håber, dette indlæg har hjulpet dig, eller i det mindste har været interessant.

**Tak for at læse med!**

**\- August Emmery**










