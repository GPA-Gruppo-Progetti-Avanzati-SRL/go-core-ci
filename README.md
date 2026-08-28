# go-core-ci

CI condivisa delle librerie **go-core**: `go-core-app`, `go-core-api`, `go-core-mongo`,
`go-core-sql`, `go-core-redis`, `go-core-batch`, `go-core-kafka`.

Due file:

- `.github/workflows/module-ci.yml` — un **reusable workflow** che ciascuna libreria richiama con un
  caller di poche righe, così la ricetta di build esiste in copia unica invece che ripetuta sette volte;
- `.golangci.yml` — la configurazione condivisa del linter, che il workflow passa a `golangci-lint`
  con `-c`. Sta qui e non nelle librerie per la stessa ragione: una copia per libreria sarebbe una
  copia da tenere allineata.

## Uso

`.github/workflows/ci.yml` nel repo della libreria:

```yaml
name: ci

on:
  push:
    branches: [master]
  pull_request:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  ci:
    uses: GPA-Gruppo-Progetti-Avanzati-SRL/go-core-ci/.github/workflows/module-ci.yml@master
```

Input opzionali:

| Input | Default | A cosa serve |
|---|---|---|
| `go-version-file` | `go.mod` | file da cui `setup-go` deduce la versione di Go |
| `cgo` | `"1"` | `CGO_ENABLED`. `go-core-kafka` è su confluent-kafka-go (CGo su librdkafka) e lo richiede |

Il workflow non riceve né usa `secrets`.

## Cosa esegue

`build` · `vet` · `gofmt` · `go test -race -count=2` · `govulncheck` · `golangci-lint` sul solo
codice nuovo.

Tre scelte che non sono ovvie:

- **`-count=2`** esegue ogni test due volte nello stesso processo e disabilita la cache: è ciò che
  smaschera lo stato globale che sopravvive fra un test e l'altro — un registro di package, un
  driver registrato in `database/sql`, una `fx.App` che non rilascia le proprie registrazioni.
- **`govulncheck` e `golangci-lint` sono compilati con `go install`** usando la toolchain di
  `setup-go`, non scaricati precompilati. Un binario precompilato è il modo in cui ci si ritrova con
  strumenti più vecchi dei sorgenti, che al primo bump di linguaggio smettono di analizzare e non lo
  dicono a nessuno. Così salgono insieme al codice, per costruzione.
- **Il linter blocca solo il codice nuovo** (`--new-from-merge-base` sulle PR,
  `--new-from-rev=HEAD~1` sui push). Le librerie hanno un debito di lint pregresso: bloccare tutto
  renderebbe la CI rossa dal primo giorno, e una CI rossa dal primo giorno viene ignorata.

Le librerie si sviluppano insieme in un workspace Go, dove ogni build usa i sorgenti locali. Questa
CI gira invece nel repo della singola libreria, quindi **senza `go.work`**: verifica che il modulo
sia consumabile da solo — cioè quello che vede chi ne fa `go get` — nel momento in cui una
discrepanza fra codice e `go.mod` verrebbe introdotta.
