# Banka-2 Infrastructure — Secrets & Security (namespace `banka-2`)

Ovaj dokument opisuje kako su tajne izdvojene iz K8s manifesta u `Secret`
resurse, sta operater mora da uradi pri deploy-u, i koji su kompromitovani
podaci koji se MORAJU rotirati.

---

## 1. Sta je promenjeno (catalog-driven fix P0-I1)

Sve plaintext tajne su uklonjene iz Deployment/StatefulSet manifesta i premestene
u K8s `Secret` resurse (template: [`secrets.yaml.example`](./secrets.yaml.example)).
Deployment-i ih sada citaju preko `valueFrom.secretKeyRef`.

| Tajna | Bila (plaintext) | Sada (Secret / key) | Fajl koji cita |
|-------|------------------|---------------------|----------------|
| `JWT_SECRET` | `trading-service.yaml` `value: dev-only-...` | `app-secrets` / `JWT_SECRET` | trading-service.yaml |
| `INTERNAL_API_KEY` | `trading-service.yaml` + `backend.yaml` `value: dev-internal-...` | `app-secrets` / `INTERNAL_API_KEY` | backend.yaml, trading-service.yaml |
| `PARTNER1_INBOUND_TOKEN` | `backend.yaml` `value: dev-outbound-...` | `app-secrets` / `PARTNER1_INBOUND_TOKEN` | backend.yaml |
| `PARTNER1_OUTBOUND_TOKEN` | `backend.yaml` `value: dev-inbound-...` | `app-secrets` / `PARTNER1_OUTBOUND_TOKEN` | backend.yaml |
| InfluxDB admin user | `influxdb.yaml` `value: admin` | `influxdb-credentials` / `INFLUX_ADMIN_USERNAME` | influxdb.yaml |
| InfluxDB admin pass | `influxdb.yaml` `value: admin12345` | `influxdb-credentials` / `INFLUX_ADMIN_PASSWORD` | influxdb.yaml |
| InfluxDB admin token | `influxdb.yaml` + `trading-service.yaml` `value: dev-token-change-me-...` | `influxdb-credentials` / `INFLUX_ADMIN_TOKEN` | influxdb.yaml, trading-service.yaml |
| RabbitMQ user | `rabbitmq.yaml` + 3 consumera `value: guest` | `rabbitmq-credentials` / `RABBITMQ_DEFAULT_USER` | rabbitmq.yaml, backend.yaml, trading-service.yaml, notification-service.yaml |
| RabbitMQ pass | isto, `value: guest` | `rabbitmq-credentials` / `RABBITMQ_DEFAULT_PASS` | isto |
| `MAIL_USERNAME` | `notification-service.yaml` plaintext | `mail-credentials` / `MAIL_USERNAME` | notification-service.yaml |
| `MAIL_PASSWORD` | `notification-service.yaml` `value: ""` (tihi SMTP fail) | `mail-credentials` / `MAIL_PASSWORD` | notification-service.yaml |

Dodatno:
- **`db.yaml`** — uklonjen `options: "SUPERUSER"` sa OBA app user-a (`db`,
  `trading-db`). Crunchy sada kreira non-superuser LOGIN role koji je vlasnik
  svoje baze (dovoljno za Hibernate `ddl-auto=update` + seed). Least privilege.
- **`seed-job.yaml`** — scrub-ovana realna Crunchy `db` SUPERUSER lozinka iz
  komentara na liniji 3 (bila committed plaintext).

> **DB lozinke (Crunchy):** NISU u `secrets.yaml.example`. PostgreSQL operator
> ih auto-generise u Secret-e `db-pguser-db` i `db-pguser-trading-db`. Deployment-i
> (`backend.yaml`, `trading-service.yaml`) i seed Job-ovi vec citaju iz njih
> preko `secretKeyRef`-a — nije bilo plaintext DB lozinke u env-u, samo u onom
> jednom komentaru.

---

## 2. Deploy workflow (OBAVEZNO pre prvog/svakog deploy-a)

Secret-i moraju postojati u klasteru PRE Deployment-a (bez njih pod ne startuje —
env var se ne resolve-uje). Dva nacina:

### Opcija A — `kubectl create secret` (preporuceno; vrednosti ne diraju disk)

```bash
# app-secrets
kubectl create secret generic app-secrets \
  --from-literal=JWT_SECRET="$(openssl rand -base64 64)" \
  --from-literal=INTERNAL_API_KEY="$(openssl rand -hex 32)" \
  --from-literal=PARTNER1_INBOUND_TOKEN="<dogovoreno sa Tim 1>" \
  --from-literal=PARTNER1_OUTBOUND_TOKEN="<dogovoreno sa Tim 1>" \
  -n banka-2

# influxdb-credentials  (token MORA biti min 32 bajta)
kubectl create secret generic influxdb-credentials \
  --from-literal=INFLUX_ADMIN_USERNAME="banka2-admin" \
  --from-literal=INFLUX_ADMIN_PASSWORD="$(openssl rand -hex 16)" \
  --from-literal=INFLUX_ADMIN_TOKEN="$(openssl rand -hex 32)" \
  -n banka-2

# rabbitmq-credentials
kubectl create secret generic rabbitmq-credentials \
  --from-literal=RABBITMQ_DEFAULT_USER="banka2" \
  --from-literal=RABBITMQ_DEFAULT_PASS="$(openssl rand -hex 16)" \
  -n banka-2

# mail-credentials  (Gmail App password, ne regular pwd — zahteva 2FA)
kubectl create secret generic mail-credentials \
  --from-literal=MAIL_USERNAME="banka2.notifications@gmail.com" \
  --from-literal=MAIL_PASSWORD="xxxx xxxx xxxx xxxx" \
  -n banka-2
```

### Opcija B — popuni template fajl

```bash
cp secrets.yaml.example secrets.yaml         # secrets.yaml je u .gitignore
# rucno zameni SVE `REPLACE_ME-*` placeholder-e pravim vrednostima
kubectl apply -f secrets.yaml -n banka-2
rm secrets.yaml                              # ne ostavljaj na disku
```

> **VAZNO:** Ako InfluxDB vec ima inicijalizovan data dir (PVC), promena
> `INFLUX_ADMIN_*` env-a NE menja postojeci token/pwd (init `setup` se izvrsava
> samo na praznom volume-u). Token u Secret-u mora biti ISTA vrednost koja je
> vec u InfluxDB-u, inace trading-service write fail-uje (401). Za rotaciju:
> rotiraj token unutar InfluxDB-a (`influx auth ...`) pa azuriraj Secret, ili
> obrisi PVC za fresh setup.

---

## 3. KOMPROMITOVANI PODACI — OBAVEZNA ROTACIJA (operativni TODO za Luku)

Sledece vrednosti su bile committed u git kao plaintext → tretirati ih kao
KOMPROMITOVANE i rotirati:

- [ ] **Crunchy `db` SUPERUSER lozinka** (vrednost je u PRE-scrub
      `seed-job.yaml:3` git-istoriji — **REDIGOVANA ovde** da se tajna ne
      re-uvodi u git preko ovog doca). **Rotirati u klasteru** —
      obrisi Crunchy user Secret da operator regenerise, npr:
      `kubectl delete secret db-pguser-db -n banka-2` (operator ga re-kreira
      sa novom lozinkom), pa restart pod-ova koji ga koriste.
- [ ] **`JWT_SECRET`** — generisi nov (`openssl rand -base64 64`); stari je
      omogucavao forge HS256 tokena. Posle rotacije svi aktivni tokeni postaju
      nevalidni (re-login).
- [ ] **`INTERNAL_API_KEY`** — nov `openssl rand -hex 32`; stari je stitio
      ceo `/internal/**` money seam.
- [ ] **InfluxDB admin token + lozinka** — nov token (`openssl rand -hex 32`)
      + pwd; stari `dev-token-change-me-32-bytes-minimum` / `admin12345`.
- [ ] **RabbitMQ user/pass** — nov dediciran user umesto `guest/guest`.
- [ ] **`PARTNER1_*` inter-bank tokeni** — re-dogovoriti sa Tim 1 ako su
      `dev-*` vrednosti ikad korisceni protiv pravog partnera.

### git-history scrub (KRITICNO)

Premestanje u Secret NE brise vrednosti iz git istorije — i dalje su dostupne u
starim commit-ima. Posle merge-a ovog fix-a, scrub-ovati istoriju sva 3 + infra
repo-a gde su tajne ikad bile (`git filter-repo` ili BFG):

> **NAPOMENA (zasto placeholder, ne literal):** ovaj doc se commituje u git
> kao remediacioni vodic. Da SAM doc ne postane novi committed leak, prava
> Crunchy SUPERUSER lozinka se NE upisuje ovde — operater je vadi iz PRE-scrub
> `seed-job.yaml:3` istorije i ubacuje lokalno tik pre pokretanja scrub-a
> (replace-text fajl je `<()` proces-substitucija, ne ostaje na disku ni u git-u).

```bash
# primer (git-filter-repo) — zameni kompromitovane stringove u CELOJ istoriji.
# <CRUNCHY_DB_SUPERUSER_PASSWORD> zameni stvarnom vrednoscu iz pre-scrub
# seed-job.yaml:3 istorije (NE komituj popunjenu verziju ovog bloka).
git filter-repo --replace-text <(cat <<'EOF'
<CRUNCHY_DB_SUPERUSER_PASSWORD>==>REDACTED
dev-only-jwt-secret-do-not-use-in-prod-rotate-via-env-var-on-each-deploy==>REDACTED
dev-internal-key-rotate-in-prod==>REDACTED
dev-token-change-me-32-bytes-minimum==>REDACTED
admin12345==>REDACTED
EOF
)
# zatim force-push (koordinisati sa timom — re-clone svima)
```

> Rotacija je VAZNIJA od scrub-a: cak i savrsen scrub ne pomaze ako je vrednost
> vec procurela. Prvo rotiraj (gore), pa scrub.

---

## 4. Verifikacija (posle izmena)

```bash
# 0 plaintext tajni u manifestima (osim komentara/placeholder-a).
# Za Crunchy SUPERUSER lozinku: zameni <CRUNCHY_PW_PREFIX> sa prvih ~8 znakova
# stvarne vrednosti iz pre-scrub istorije (NE komituj popunjenu verziju).
grep -rn "dev-only\|dev-internal\|admin12345\|dev-token-change-me\|<CRUNCHY_PW_PREFIX>" *.yaml
#   -> nema pogodaka

# svi secret env-ovi koriste secretKeyRef (nema `value:` na njima):
grep -rn "value: guest\|value: admin12345" *.yaml   # -> nema pogodaka

# YAML/struktura validna:
for f in *.yaml; do kubectl apply --dry-run=client -f "$f"; done
```
