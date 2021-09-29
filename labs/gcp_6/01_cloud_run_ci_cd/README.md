# Cloud Run + CI/CD

## Założenia wstępne

- [X] Istnieje aplikacja Node.js (jeden endpoint `/greet?name=Mac`)
- [X] Aplikacja spakowana jest w kontener

## Uruchomienie aplikacji w Cloud Run

- [X] Uruchomienie usługi na [lokalnym emulatorze Cloud Run](https://cloud.google.com/run/docs/testing/local#cloud-sdk)

   ```bash
   gcloud alpha code dev
   ```

   Lokalne środowisko hostowane jest na lokalnym `minikube` na maszynie programisty. Możemy zobaczyć detale środowiska wykonując `kubectl get all --all-namespaces`. Po uruchomieniu środowiska dodawany jest nowy wpis do `~/.kube/config` (wpis jest usuwany po zastopowaniu środowiska).

- [X] Uruchomienie usługi na Cloud Run zbudowanej na podstawie [lokalnej kopii kodu źródłowego](https://cloud.google.com/run/docs/deploying-source-code)

   ```bash
   gcloud alpha run deploy testy --source . --platform managed --region europe-west3 --allow-unauthenticated
   ```

   Usługa budowana jest na podstawie `Dockerfile` z folderu z aplikacją. Zbudowany obraz przechowywany jest w Container Registry (repozytorium: `cloud-run-source-deploy`). Usługa uruchamiana jest na podstawie obrazu z Container Registry.

- [X] Wylistowanie szczegółów usługi

   ```bash
   gcloud run services list
   ```

   ```bash
   gcloud run services descirbe [NAME]
   ```

## Uruchomienie aplikacji z obrazu w Container Registry

- [X] Zbuduj obraz aplikacji lokalnie i udostępnij go w repozytorium na gcloud

   ```bash
   docker build -t lpw9:latest .
   docker tag lpw9:latest eu.gcr.io/training-w9-lpstudentXX/lpw9:0.1.0
   docker push eu.gcr.io/training-w9-lpstudentXX/lpw9:0.1.0
   gcloud container images list --repository eu.gcr.io/training-w9-lpstudentXX
   gcloud container images list-tags eu.gcr.io/training-w9-lpstudentXX/lpw9
   ```

   Container Registry jest prywatnym rejestrem obrazów.

- [X] Wdróż nową wersję aplikacji na Cloud Run (z obrazu przechowywanego na Container Registry)

   ```bash
   gcloud run deploy testy --image eu.gcr.io/training-w9-lpstudentXX/lpw9:0.1.0 --platform managed --region europe-west3 --allow-unauthenticated
   ```

## Budowanie obrazu kontenera za pomocą Cloud Build

- [X] Lokalnie rozwijana wersja aplikacji zostanie zbudowana za pomocą Cloud Build

  ```bash
  gcloud builds submit --tag eu.gcr.io/training-w9-lpstudentXX/lpw9:0.2.0
  ```

  W ten sposób nie musisz ręcznie tagować obrazów kontenera (za pomocą Docker CLI).

## Aplikacja budowana jest automatycznie po każdym push do branch `master`

- [X] Kod aplikacji udostępniony jest w Code Repository

  ```bash
  gcloud source repos create lpw9
  gcloud source repos clone lpw9 # poza repozytorium `lp-workshops`
  ```

- [X] Cloud Build wykonuje build aplikacji po każdym push do brancha `master`. Wynikiem build jest obraz kontenera udostępniony w Container Registry

  ```bash
  touch cloudbuild.yaml

  gcloud beta builds triggers create cloud-source-repositories --name="master-after-push-build-docker-image-eu" --repo="lpw9" --branch-pattern="master" --build-config=cloudbuild.yaml
  ```

- [X] Nowa wersja aplikacji jest automatycznie deployowana gdy nowy obraz kontenera będzie dostępny

  W ustawieniach Cloud Run dla Service Account włącz uprawnienia Cloud Run Admin. W popup potwierdź nadanie uprawnień dla kont Service Account.

## Aplikacja łączy się z Cloud SQL

- [X] Istnieje instancja Cloud SQL (schema "actors")

  ```bash
  gcloud sql instances create sql-instance --database-version=POSTGRES_13 --cpu=1 --memory=3840MiB --zone=europe-west3-a --root-password=password123
  gcloud sql connect sql-instance --user=postgres
  CREATE DATABASE lp_test;
  \c lp_test

  CREATE TABLE winners (
    wid SERIAL PRIMARY KEY,
    year INTEGER,
    age INTEGER,
    name varchar(500),
    movie varchar(500)
  );

  gcloud sql import csv sql-instance gs://lpw9-x/data.csv --database=lp_test --table=winners
  ```

- [X] Cloud Run łączy się z Cloud SQL za pomocą credentials pobranych z Secret Manager

  > Google recommends that you use Secret Manager to store sensitive information such as SQL credentials. You can pass secrets as environment variables or mount as a volume with Cloud Run.

  Dodaj nowy sekret o nazwie INSTANCE_CONNECTION_NAME w [Secret Manager](https://console.cloud.google.com/security/secret-manager). Do sekretu dodaj role `Secret Manager Secret Accessor` dla wybranego Service Account (`gcloud run services describe <SERVICE_NAME>`).

  Pobierz wartość `INSTANCE_CONNECTION_NAME`:

  ```bash
  gcloud sql instances describe sql-instance --format="table(connectionName)"
  ```

  Przejdź do [Cloud Run](https://console.cloud.google.com/run/). Kliknij Wykonaj nowy deploy. Skonfiguruj wartości (instancja Cloud SQL oraz Secret)

- [X] Cloud Run pobiera dane z bazy (endpoint `GET /winners`)


## Zwiększenie security

- [X] Obrazy do Container Registry może pushować tylko Code Repository

  Odbierz nie dziedziczone uprawnienia do Cloud Storage Bucket z artefaktami. Pozostaw uprawnienia dla Service Accounts (nawet jeśli nie są dziedziczone).

## Monitoring

- [X] Debug

  Przejdź do Code Repository, wybierz plik `index.js` i postaw breakpoint w wybranym miejscu (np. na linii `await client.connect();`). Odśwież stronę.

## Aplikacja wdrażana jest na środowisko `prod`

- Istnieje środowisko `prod` na którym działa aplikacja
- Nowe wersje aplikacji są wdrażane na środowisko `prod` na żądanie
- Aplikacja łączy się z inną instancją bazy danych (connection string jest przechowywany w Secret Manager)
