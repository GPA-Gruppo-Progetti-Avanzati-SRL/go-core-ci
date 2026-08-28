# go-core-ci

CI condivisa dei moduli **go-core** (`go-core-app`, `go-core-api`, `go-core-mongo`, `go-core-sql`,
`go-core-redis`, `go-core-batch`, `go-core-kafka`).

Contiene un solo file: `.github/workflows/module-ci.yml`, un **reusable workflow** che ciascun modulo
richiama con un caller di sei righe:

```yaml
jobs:
  ci:
    uses: GPA-Gruppo-Progetti-Avanzati-SRL/go-core-ci/.github/workflows/module-ci.yml@main
```

## Perché un repo a sé

I sette moduli sono repository **pubblici**; il monorepo ombrello `go-core`, dove vive il resto
dell'automazione (`check-drift.sh`, `drift.yml`, `static.yml`), è **privato**. Un caller pubblico non
risolve un reusable workflow ospitato in un repo privato, quindi il workflow condiviso non poteva
stare lì. L'alternativa era copiarlo sette volte — e sette copie sono sette copie che divergono.

Qui non ci sono segreti: è una ricetta di build, e il workflow non riceve né usa `secrets`.

## Cosa esegue

`build` · `vet` · `gofmt` · `go test -race -count=2` · `govulncheck` · `golangci-lint` sul solo
codice nuovo. Gira nel repo del modulo, quindi **senza `go.work`**: è la verifica che il modulo sia
consumabile da solo, fatta nel momento in cui una deriva verrebbe introdotta.

`-count=2` è deliberato: esegue ogni test due volte nello stesso processo e smaschera lo stato
globale fra un test e l'altro.

Gli strumenti sono compilati con `go install` usando la toolchain di `setup-go`, non scaricati
precompilati: un binario precompilato è il modo in cui ci si ritrova con un linter più vecchio dei
sorgenti.
