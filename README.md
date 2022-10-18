# Infrastruktur som kode med Terraform og AWS App runner

Når du er ferdig med denne oppgaven vil du ha et repository som inneholder en Spring Boot applikasjon.
Når du gjør en commit på main branch i dette, så vil GitHub actions gjøre

* Lage et Docker Image, og Push til ECR. Både med "latest" tag - og med en spesifikk  tag som matcher git commit.
* Bruke Terraform til å lage AWS infrastruktur, IAM roller og en AWS App Runner Service.

* I denne oppgaven skal vi gjøre en en docker container tilgjengelig over internett ved hjelp av en tjeneste AWS Apprunner.
* Apprunner lager nødvendig infrastruktur for containeren, slik at du som utvikler kan fokusere på koden.

Vi skal også se nærmere på mer avansert GitHub Actions: For eksempel;

* To jobber og avhengigheter mellom disse.
* En jobb vil lage infrastruktur med terraform, den andre bygge Docker container image
* Bruke terraform i Pipeline - GitHub actions skal kjøre Terraform for oss.
* En liten intro til AWS IAM og Roller

## Lag en fork

Du må start emd å lage en fork av dette repoet til din egen GitHubkonto.

![Alt text](img/fork.png  "a title")

## Logg i Cloud 9 miljøet ditt

![Alt text](img/aws_login.png  "a title")

* Logg på med din AWS bruker med URL, brukernavn og passord gitt i klassrommet
* Gå til tjenesten Cloud9 (Du nå søke på Cloud9 uten mellomrom i søket)
* Velg "Open IDE"
* Hvis du ikke ser ditt miljø, kan det hende du har valgt feil region. Hvilken region du skal bruke vil bli oppgitt i klasserommet.

### Lag et Access Token for GitHub

* Når du skal autentisere deg mot din GitHub konto fra Cloud 9 trenger du et access token.  Gå til  https://github.com/settings/tokens og lag et nytt.
* NB. Ta vare på tokenet et sted, du trenger dette senere når du skal gjøre ```git push```

![Alt text](img/generate.png  "a title")

Access token må ha "repo" tillatelser, og "workflow" tillatelser.

![Alt text](img/new_token.png  "a title")

### Lage en klone av din Fork (av dette repoet) inn i ditt Cloud 9 miljø

Fra Terminal i Cloud 9. Klone repositoriet *ditt* med HTTPS URL.

```
git clone https://github.com/≤github bruker>/terraform-app-runner.git
```

Får du denne feilmeldingen ```bash: /terraform-app-runner: Permission denied``` - så glemte du å bytte ut <github bruker> med
ditt eget Github brukernavn :-)

![Alt text](img/clone.png  "a title")

OBS Når du gjør ```git push``` senere og du skal autentisere deg, skal du bruke GitHub  brukernavn, og access token som passord,

For å slippe å autentisere seg hele tiden kan man få git til å cache nøkler i et valgfritt antall sekunder på denne måten;

```shell
git config --global credential.helper "cache --timeout=86400"
```

Konfigurer også brukernavnet og e-posten din for GitHub CLI. Da slipepr du advarsler i terminalen når du gjør commit senere.

````shell
git config --global user.name <github brukernavn>
git config --global user.email <email for github bruker>
````

## Slå på GitHub actions for din fork

I din fork av dette repositoriet, velg "actions" for å slå på støtte for GitHub actions i din fork.

![Alt text](img/7.png "3")

### Sett Repository secrets

* Lag AWS IAM Access Keys for din bruker
* Vi sette hemmeligheter ved å legge til følgende kodeblokk i github actions workflow fila vår slik at terraform kan autentisere seg med vår identitet, og våre rettigheter.

```yaml
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_REGION: eu-west-1
```

### Test terraform, med lokal state i Cloud 9

Gå til terraform-demo mappen i dett repoet med
```
cd terraform-demo
```

* En terraform state-fil er bindeleddet mellom den faktiske infrastrukturen og infrastruktur-koden.
* En terraform backend er et sted som å lagre terraform state. Det finnes mange implementasjoner, for eksempel S3
* Hvis du ikke deklarerer en backend i koden, vil terraform lage en state-fil på din maskin, i samme katalog som du
  kjører terraform fra.

* Gå til s3-demo katalogen.
* Endre s3.tf og endre bucket navnet slik at det blir globalt unikt.

Deretter utfører du kommandoene

 ```
terraform init 
terraform plan
terraform apply
```  

* Hvis du slår på visning av skjulte filer i Cloud9 vil du nå se en ````.terraform```` katalog. Denne inneholder blant annet kode for
  terraform "provider" for AWS (Det som gjør at Terraform kan lage-, endre og slette infrastruktur i AWS) - Disse filene ble lastet ned på ```terraform init``
* Når apply er ferdig, vil du se en terraform.tfstate fil i katalogen du kjørte terrafomr fra. Se å filen.
* Du kan nå forsøke å slette denne filen, og kjøre terraform apply en gang til. Terraform prøver å opprette bucketen på nytt,
  fordi den ikke lenger vet at denne er oppprettet av Terraform. Denne informasjonen ligger i "state" filen til terraform som du nettopp slettet!

* Gå til Amazon S3 i AWS kontoen din, og slett bucketen.

Endre provider.tf ved å legge på en _backend_ blokk, slik at den ser omtrent slik ut

```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "4.33.0"
    }
  }
 backend "s3" {
    bucket = "pgr301-2021-terraform-state"
    key    = "<student-navn>/apprunner-demo.state"
    region = "eu-north-1"
  }
}
```

* Her forteller vi Terraform at state-informasjon skal lagres i S3, i en Bucket som ligger i Stockholm regionen, med et filnavn du selv bestemmer
  ved å endre "key"

for å starte med blank ark må du fjerne terraform.tf og hele .terraform katalogen

Deretter utfører du kommandoene

 ```
terraform init 
terraform plan
terraform apply
```  

Legg merke til at du nå ikke har noe state fil i Cloud9, men se i S3 at du har fått en fil i buckenten som heter; pgr301-2021-terraform-state
og med objektnavnet du valgte.

## Terraform med GitHub actions

Du skal ikke bruke provider.tf i *terraform-demo* katalogen for denne oppgaven!

### Sett Repository secrets

* Lag AWS IAM Access Keys for din bruker. NB Du må gjøre dette på nytt, hvis du har gjort dette før.
* Vi setter hemmeligheter ved å legge til følgende kodeblokk i github actions workflow fila vår slik at terraform kan autentisere seg med vår identitet, og våre rettigheter.

```yaml
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_REGION: eu-west-1
```

Vi skal nå se på hvordan vi kan få GitHub actions til å kjøre Terraform for oss. Det er par nye nyttige elementer i pipelinen.

Her ser vi et steg i en pipeline med en ```if``` - som bare skjer dersom det er en ```pull request``` som bygges, vi ser også at
pipeline får lov til å _fortsette dersom dette steget feiler.

```yaml
  - name: Terraform Plan
    id: plan
    if: github.event_name == 'pull_request'
    run: terraform plan -no-color
    continue-on-error: true
```

Når noen gjør en Git push til *main* branch, kjører vi ```terraform apply``` med ett flag ```--auto-approve``` som gjør at terraform ikke
spør om lov før den kjører.

```yaml
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve
```

Terraform trenger docker container som lages i en egen GitHub Actions jobb. Vi kan da bruke ```needs``` for å lage en avhengighet mellom en eller flere jobber;

```yaml
  terraform:
    needs: build_docker_image
```

## Endre provider.tf
Provider.tf filen inneholder konfigurasjon for terraform, blant annet informasjon om hvor Terraform kan finne en *state fil*
Endre key, til ditt eget studentnavn.

```hcl
backend "s3" {
    bucket = "pgr301-2021-terraform-state"
    key    = "<key>/apprunner.state"
    region = "eu-north-1"
}
```

## Finn ditt eget ECR repository

* Det er laget et ECR repository til hver student som en del av labmiljøet
* Dette heter *studentnavn-private*
* Gå til tjenesten ECR og sjekk at dette finnes

## Gjør nødvendig endringer i pipeline.yml

Som dere ser er "glenn" hardkodet ganske mange steder, bruk ditt eget ECR repository.

* Oppgave: Endre kodeblokken under slik at den også pusher en "latest" tag.
* Husk at det må være samsvar mellom tag du lager med _docker build_ og det du bruker i _docker tag_ kommandoen.

```sh
  docker build . -t hello
  docker tag hello 244530008913.dkr.ecr.eu-west-1.amazonaws.com/glenn:$rev
  docker push 244530008913.dkr.ecr.eu-west-1.amazonaws.com/glenn:$rev
```

## Endre terraform apply linjen

Finn denne linjen, du må endre prefix til å være ditt studentnavn, du må også legge inn studentnavn i image variabelen
for å fortelle app runner hvilket container som skal deployes.

```
 run: terraform apply -var="prefix=<studentnavn>" -var="image=244530008913.dkr.ecr.eu-west-1.amazonaws.com/<studentnavn>-private:latest" -auto-approve
```

## Test

* Kjør byggejobben manuelt førte gang gang.  Det vil det lages en docker container som pushes til ECR repository. App runner vil lage en service
* Sjekk at det er dukket opp to container images i ECR. En med en tag som matcher git commit, og en som heter latest.