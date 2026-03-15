# Pelias Geocoder — Germany Deployment Guide

Deploy the plan4better custom Pelias geocoder with pre-built data from Hetzner S3.

## Prerequisites

- Docker & Docker Compose
- AWS CLI (`pip install awscli` or `apt install awscli`)
- ~30 GB free disk space
- `vm.max_map_count` ≥ 262144 (for Elasticsearch)

```bash
# Check and set vm.max_map_count (required for ES)
sysctl vm.max_map_count
# If < 262144:
sudo sysctl -w vm.max_map_count=262144
# To persist across reboots:
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

## 1. Clone the Docker repo

```bash
git clone git@github.com:plan4better/docker.git pelias
cd pelias/projects/germany
```

## 2. Configure environment

Create/edit `.env`:

```env
COMPOSE_PROJECT_NAME=pelias
DATA_DIR=./data
DOCKER_USER=1000:1000
```

> **Note:** Set `DOCKER_USER` to your current user's UID/GID (run `id -u` and `id -g`).
> This ensures containers can read/write the data directory.

## 3. Download pre-built data from Hetzner S3

The data (~23 GB) contains the pre-built Elasticsearch index, interpolation databases,
OpenStreetMap extracts, OpenAddresses, Who's On First, polylines, and placeholder data.

```bash
mkdir -p data

export AWS_ACCESS_KEY_ID="<hetzner-access-key>"
export AWS_SECRET_ACCESS_KEY="<hetzner-secret-key>"

aws --endpoint-url https://nbg1.your-objectstorage.com \
    s3 sync s3://goat-dev/data/pelias/ data/ \
    --multipart-chunksize 100MB
```

Verify the download:

```bash
du -sh data/*
# Expected:
# 12G   data/elasticsearch
# 3.3G  data/interpolation
# 1.2G  data/openaddresses
# 5.4G  data/openstreetmap
# 112M  data/placeholder
# 166M  data/polylines
# 1.1G  data/whosonfirst
```

## 4. Build custom images

The API and schema use plan4better forks with German-specific improvements:

```bash
docker compose build api schema
```

This builds from:
- `plan4better/api` — German address normalizer, fuzzy street matching, abbreviation expansion
- `plan4better/schema` — German diacritics transliteration (ue→u, oe→o, ae→a)

The API build runs the full test suite (5587 tests) as part of the Docker build.

## 5. Start services

```bash
# Start Elasticsearch first and wait for it to be healthy
docker compose up -d elasticsearch
# Wait ~15 seconds, then verify:
curl -s http://localhost:9200/_cluster/health | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])"
# Should print "green"

# Start all remaining services
docker compose up -d api libpostal placeholder interpolation pip
```

## 6. Verify

Wait ~30 seconds for all services to initialize, then:

```bash
# Check all containers are running
docker compose ps

# Test a geocoding query
curl -s "http://localhost:4000/v1/search?text=Hauptstr+34+79576+Weil+am+Rhein" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
f = d['features'][0]
p = f['properties']
print(f\"{p['name']} | {p.get('street','')} | {p.get('locality','')} | {p.get('postalcode','')} | layer={p['layer']}\")
"
# Expected: Hauptstraße 34 | Hauptstraße | Weil am Rhein | 79576 | layer=address
```

## Service Ports

| Service         | Port | Visibility |
|-----------------|------|------------|
| API             | 4000 | Public (`0.0.0.0`) |
| Elasticsearch   | 9200 | Localhost only |
| Placeholder     | 4100 | Localhost only |
| PIP             | 4200 | Localhost only |
| Interpolation   | 4300 | Localhost only |
| Libpostal       | 4400 | Localhost only |

## Stopping / Restarting

```bash
# Stop all
docker compose down

# Restart (data persists in ./data)
docker compose up -d elasticsearch
# Wait for ES green, then:
docker compose up -d api libpostal placeholder interpolation pip
```

## Rebuilding from scratch (optional)

If you ever need to re-import data instead of using the pre-built S3 snapshot:

```bash
# 1. Download raw source data
docker compose run --rm openstreetmap   # downloads OSM PBF
docker compose run --rm openaddresses   # downloads OA CSVs
docker compose run --rm whosonfirst     # downloads WOF data
docker compose run --rm polylines       # generates street polylines

# 2. Create the ES index schema
docker compose run --rm schema ./bin/create_index

# 3. Import data into ES
docker compose run --rm openstreetmap ./bin/start
docker compose run --rm openaddresses ./bin/start
docker compose run --rm csv-importer ./bin/start
docker compose run --rm polylines ./bin/start

# 4. Build interpolation and placeholder databases
docker compose run --rm interpolation ./bin/build
docker compose run --rm placeholder ./cmd/extract.sh
docker compose run --rm placeholder ./cmd/build.sh
```

## GitHub Repositories

| Repo | Purpose |
|------|---------|
| [plan4better/docker](https://github.com/plan4better/docker) | Docker Compose config (`projects/germany/`) |
| [plan4better/api](https://github.com/plan4better/api) | Custom API with German normalizers |
| [plan4better/schema](https://github.com/plan4better/schema) | ES schema with diacritics transliteration |
| [plan4better/query](https://github.com/plan4better/query) | Query layer: fuzzy streets, postcode boost |
| [plan4better/sorting](https://github.com/plan4better/sorting) | Result ranking: PLZ/city mismatch penalties |

> `query` and `sorting` are npm dependencies of `api` (referenced in `package.json`).
> They are pulled automatically during `docker compose build api`.
