# <img src=".gitlab/ansible.png" alt="Ansible" height="30"/> Ansible - klaster HashiCorp Vault

Playbook Ansible przygotowujący trzywęzłowy klaster HashiCorp Vault wraz z systemem operacyjnym, certyfikatami TLS, HAProxy oraz wirtualnym adresem IP zarządzanym przez Keepalived.

::include{file=.gitlab/badges.md}

---

## Zakres projektu

Playbook `playbooks/install.yml` wykonuje następujące operacje na hostach z grupy `vault`:

- konfiguruje strefę czasową, locale oraz nazwę hosta,
- zarządza kontami użytkowników, grupami i uprawnieniami `sudo`,
- utwardza konfigurację SSH,
- instaluje wymagane pakiety i zaufane certyfikaty CA,
- instaluje oraz konfiguruje HashiCorp Vault,
- konfiguruje magazyn danych Integrated Storage (Raft),
- uruchamia HAProxy w trybie TCP passthrough,
- konfiguruje Keepalived i wspólny adres VIP,
- instaluje lokalny mechanizm automatycznego unseal oraz okresowej rekonfiguracji.

## Architektura

| Host | Adres IP | Rola Keepalived |
| --- | --- | --- |
| `vault-1005.rachuna.dev` | `10.3.2.5` | MASTER |
| `vault-1006.rachuna.dev` | `10.3.2.6` | BACKUP |
| `vault-1007.rachuna.dev` | `10.3.2.7` | BACKUP |

Wirtualny adres klastra to `10.3.2.254/24`. HAProxy nasłuchuje na porcie `443` i przekazuje ruch TLS do aktywnego, odpieczętowanego węzła Vault na porcie `8200`. Statystyki HAProxy są dostępne na porcie `8404` pod ścieżką `/stats`.

Vault działa z włączonym TLS i interfejsem UI. Domyślnym backendem storage jest Raft; konfiguracja Consul jest dostępna w roli, ale pozostaje wyłączona.

## Wymagania

Kontroler Ansible musi mieć:

- dostęp SSH do wszystkich hostów z inventory,
- użytkownika z możliwością użycia `become`,
- Ansible oraz kolekcję `community.hashi_vault`,
- dostęp do repozytoriów GitLab wskazanych w `requirements.yml`,
- dostęp do Vault przechowującego sekrety używane przez inventory,
- poprawne rekordy DNS dla węzłów i adresu `vault.rachuna.dev`.

Hosty docelowe muszą używać wspieranego systemu z rodziny Debian, RedHat lub Alpine i udostępniać interpreter Python wskazany w `inventory/host_vars/`.

## Przygotowanie

1. Zainstaluj kolekcję i role Ansible:

   ```bash
   ansible-galaxy collection install community.hashi_vault
   ansible-galaxy role install -r requirements.yml -p playbooks/roles
   ```

2. Ustaw zmienne środowiskowe wymagane przez inventory i lookupy Vault:

   ```bash
   export ANSIBLE_USER="tech_user"
   export ANSIBLE_PASSWORD="<haslo-become>"
   export VAULT_ADDR="https://vault.example.org"
   export VAULT_TOKEN="<token>"
   ```

   W środowisku z prywatnym CA ustaw również `REQUESTS_CA_BUNDLE` na plik zawierający zaufany łańcuch certyfikatów. Projekt zawiera `.envrc`, który może być ładowany przez `direnv`; nie należy umieszczać w nim sekretów przeznaczonych do commitowania.

3. Zweryfikuj wartości w:

   - `inventory/hosts.yml` oraz `inventory/host_vars/`,
   - `inventory/group_vars/all/` dla kont, certyfikatów, locale i SSH,
   - `inventory/group_vars/vault/` dla Vault, HAProxy, Keepalived i repozytoriów.

Sekrety są pobierane przez lookup `community.hashi_vault.vault_kv2_get`. Ścieżki i mount pointy znajdują się w `inventory/group_vars/all/secrets.yml`.

## Uruchomienie

Najpierw sprawdź łączność z hostami:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/test_connection.yml
```

Sprawdź składnię playbooka:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/install.yml --syntax-check
```

Uruchom wdrożenie:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/install.yml
```

Zakres wykonania można ograniczyć standardowymi opcjami Ansible, na przykład do jednego hosta:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/install.yml --limit vault-1005
```

## Weryfikacja

Po wdrożeniu sprawdź stan usług na węzłach:

```bash
systemctl status vault vault-unseal.service vault-reconfigure.timer haproxy keepalived
```

Stan Vault można zweryfikować przez API klastra:

```bash
curl --cacert /etc/vault.d/ca.crt https://vault.rachuna.dev/v1/sys/health
```

## Struktura repozytorium

```text
.
├── inventory/
│   ├── group_vars/            # konfiguracja wspólna i grupy vault
│   ├── host_vars/             # parametry poszczególnych węzłów
│   └── hosts.yml              # inventory klastra
├── playbooks/
│   ├── roles/
│   │   ├── install-vault/     # instalacja i konfiguracja Vault
│   │   └── vault-auto-unseal/ # lokalny auto-unseal i rekonfiguracja
│   ├── install.yml            # główny playbook wdrożeniowy
│   └── test_connection.yml    # test dostępu do hostów
└── requirements.yml           # zewnętrzne role Ansible
```

## Bezpieczeństwo

Rola `vault-auto-unseal` zapisuje klucz unseal na każdym węźle w pliku `/etc/vault.d/unseal_keys.json`. Jest to rozwiązanie przeznaczone wyłącznie dla środowiska deweloperskiego i nie powinno być używane w produkcji. W środowisku produkcyjnym należy zastosować wspierany mechanizm auto-unseal, np. KMS, HSM lub Transit Secrets Engine.

Nie zapisuj tokenów Vault, haseł, kluczy unseal ani prywatnych kluczy SSH w repozytorium. Sekrety powinny być przekazywane przez bezpieczne zmienne środowiskowe albo pobierane z dedykowanego magazynu sekretów.

## Uwagi

- Rola `install-vault` domyślnie instaluje Vault `1.19.0`.
- Konfiguracja wymaga dokładnie jednego backendu storage: Raft albo Consul.
- Pierwsza inicjalizacja klastra Vault i dołączanie kolejnych węzłów Raft nie są wykonywane przez główny playbook.
- Generowanie certyfikatów jest obecnie wyłączone przez `inv_enabled_generate_certificates: false`; wymagane pliki TLS muszą istnieć przed uruchomieniem Vault.

---

::include{file=.gitlab/footer.md}
