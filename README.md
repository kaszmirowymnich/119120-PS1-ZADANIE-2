# Model językowy (llamafile) jako usługa systemd

Uruchomienie gotowego modelu **Qwen3.5-4B (Q5_K_S)** w postaci pojedynczego pliku `llamafile` jako usługi `systemd` na Ubuntu (ARM64, UTM na macOS). API REST jest dostępne tylko z gospodarza przez adres host-only.

## Środowisko

| Element | Wartość |
|---|---|
| Hypervisor | UTM (QEMU), macOS |
| System gościa | Ubuntu, `aarch64` |
| RAM maszyny | 6 GB (ok. 5,3 GiB widoczne w systemie), 2 wątki CPU |
| Karta NAT | `enp0s1`, `10.0.2.15` |
| Karta host-only | `enp0s2`, `192.168.128.2` |
| Adres hosta (Mac) | `192.168.128.1` (`bridge100`) |
| Model | `Qwen3.5-4B-Q5_K_S.llamafile` (3 910 943 838 B) |
| Wersja llamafile | v0.10.6 |

## Kroki (do powtórzenia na nowej maszynie)

Poniżej `HOSTONLY_IP` oznacza adres karty host-only maszyny (u mnie `192.168.128.2`). Sprawdzisz go poleceniem `ip -br a`.

### 1. Użytkownik systemowy i katalogi

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin llamafile
sudo mkdir -p /opt/llamafile
sudo mkdir -p /var/lib/llamafile
sudo chown llamafile:llamafile /var/lib/llamafile
```

Katalog `/var/lib/llamafile` jest potrzebny, bo użytkownik nie ma katalogu domowego, a llamafile zapisuje w nim pliki robocze. Bez niego pojawia się błąd `mkdir: cannot create directory '/home/llamafile': Permission denied`.

### 2. Pobranie modelu przez curl

Pobieramy od razu do `/opt/llamafile`. Katalog `/tmp` bywa małym systemem plików w pamięci i pobieranie kończy się błędem `curl: (23) Failure writing output`.

```bash
sudo curl -fL -C - -o /opt/llamafile/model.llamafile \
  https://huggingface.co/mozilla-ai/llamafile_0.10/resolve/main/Qwen3.5-4B-Q5_K_S.llamafile
sudo chown root:llamafile /opt/llamafile/model.llamafile
sudo chmod 755 /opt/llamafile/model.llamafile
ls -l /opt/llamafile/
```

Flaga `-C -` pozwala wznowić przerwane pobieranie. Rozmiar końcowy powinien wynosić 3910943838 bajtów (można porównać z `curl -sIL <url> | grep -i content-length`).

### 3. Test ręczny

```bash
sudo -u llamafile env HOME=/var/lib/llamafile /opt/llamafile/model.llamafile --help | head -40
sudo -u llamafile env HOME=/var/lib/llamafile /opt/llamafile/model.llamafile \
  --server --gpu disable --host HOSTONLY_IP --port 8080
```

W logu pojawia się linia `listening on http://HOSTONLY_IP:8080`. Zatrzymanie: `Ctrl+C`.

### 4. Zapora (ufw)

Jeśli `ufw` jest aktywny, domyślnie blokuje port 8080 i zapytania z gospodarza kończą się timeoutem. Otwieramy port wyłącznie dla podsieci host-only:

```bash
sudo ufw status
sudo ufw allow from 192.168.128.0/24 to any port 8080 proto tcp
sudo ufw status numbered
```

Stan reguł po konfiguracji:

```text
     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    192.168.128.0/24
[ 2] 8080/tcp                   ALLOW IN    192.168.128.0/24
```

### 5. Plik jednostki systemd

Plik `/etc/systemd/system/llamafile.service` (kopia w `config/llamafile.service`):

```ini
[Unit]
Description=Llamafile LLM server
After=network-online.target
Wants=network-online.target

[Service]
User=llamafile
Group=llamafile
Environment=HOME=/var/lib/llamafile
ExecStart=/bin/sh /opt/llamafile/model.llamafile --server --gpu disable --host 192.168.128.2 --port 8080
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Uzasadnienie wybranych ustawień:

- `User=llamafile` – usługa działa jako osobny użytkownik systemowy, nie jako root.
- `ExecStart=/bin/sh ...` – llamafile jest plikiem w formacie APE (Cosmopolitan), którego systemd nie uruchamia bezpośrednio. Bez `/bin/sh` usługa kończy się błędem `status=203/EXEC`.
- `Restart=always` – po zabiciu procesu (`kill` wysyła SIGTERM, który kończy proces czysto) `Restart=on-failure` mogłoby usługi nie wznowić.
- `Environment=HOME=...` – patrz krok 1.
- `--host 192.168.128.2` – serwer nasłuchuje tylko na adresie host-only.
- `--gpu disable` – maszyna wirtualna nie ma GPU.
- `WantedBy=multi-user.target` – autostart po restarcie systemu.

### 6. Włączenie usługi

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now llamafile
systemctl status llamafile --no-pager
journalctl -u llamafile -f
```

## Weryfikacja wymagań

### Usługa działa jako użytkownik `llamafile`, nie root

Proces jest uruchamiany z `User=llamafile` (patrz plik jednostki). Proces w `systemctl status` to `/var/lib/llamafile/.ape-1.10` (llamafile rozpakowuje własny interpreter do `HOME` użytkownika usługi).

### Nasłuch tylko na adresie host-only (`ss -tlnp`)

```bash
sudo ss -tlnp
```

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:PortProcess
LISTEN 0      128    192.168.128.2:8080      0.0.0.0:*    users:((".ape-1.10",pid=3199,fd=3))
LISTEN 0      4096      127.0.0.54:53        0.0.0.0:*    users:(("systemd-resolve",pid=570,fd=19))
LISTEN 0      4096         0.0.0.0:22        0.0.0.0:*    users:(("sshd",pid=1473,fd=3),("systemd",pid=1,fd=191))
LISTEN 0      4096   127.0.0.53%lo:53        0.0.0.0:*    users:(("systemd-resolve",pid=570,fd=17))
LISTEN 0      4096            [::]:22           [::]:*    users:(("sshd",pid=1473,fd=4),("systemd",pid=1,fd=193))
```

Port 8080 nasłuchuje wyłącznie na `192.168.128.2`, a nie na `0.0.0.0`. Port 22 (SSH) jest dodatkowo ograniczony zaporą `ufw` do podsieci host-only.

### Zapytanie z gospodarza i odpowiedź modelu

Wykonane na macOS:

```bash
curl http://192.168.128.2:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"local","messages":[{"role":"user","content":"Powiedz cześć w jednym zdaniu."}]}'
```

Odpowiedź (model odpowiada `Cześć!`, ok. 12 tokenów/s):

```json
{"choices":[{"finish_reason":"stop","index":0,"message":{"role":"assistant","content":"Cześć!"}}],"created":1793027420,"model":"/zip/Qwen3.5-4B-Q5_K_S.gguf","system_fingerprint":"b1789468725-0177dcc73","object":"chat.completion","usage":{"completion_tokens":5,"prompt_tokens":23,"total_tokens":28,"prompt_tokens_details":{"cached_tokens":0}},"id":"chatcmpl-IBjgmsP8XPWmLNQtPCwqNzijyFpRAa7a","timings":{"cache_n":0,"prompt_n":23,"prompt_ms":1039.294,"prompt_per_token_ms":45.18669565217392,"prompt_per_second":22.130407757573888,"predicted_n":5,"predicted_ms":329.305,"predicted_per_token_ms":82.32625,"predicted_per_second":12.146794005557158}}
```

### Brak dostępu przez pozostałe adresy

Na maszynie wirtualnej (adres pętli zwrotnej i adres karty NAT):

```bash
curl -m 3 http://127.0.0.1:8080/health; echo "kod: $?"
curl -m 3 http://10.0.2.15:8080/health; echo "kod: $?"
```

```text
curl: (7) Failed to connect to 127.0.0.1 port 8080 after 0 ms: Could not connect to server
kod: 7
curl: (7) Failed to connect to 10.0.2.15 port 8080 after 0 ms: Could not connect to server
kod: 7
```

Oba połączenia są odrzucone (kod 7). Dodatkowo `ufw` przepuszcza port 8080 wyłącznie z podsieci `192.168.128.0/24`.

### Zużycie pamięci

Fragment `systemctl status llamafile` po uruchomieniu usługi:

```text
Memory: 1.8G (peak: 1.8G, swap: 43.3M, swap peak: 44M)
```

Po restarcie maszyny, po kilkudziesięciu sekundach pracy:

```text
Memory: 4.6G (peak: 4.7G, swap: 243.6M, swap peak: 244.9M)
```

Wartość z `systemctl status` obejmuje także strony pliku modelu zmapowane do pamięci (cache), dlatego rośnie w miarę czytania wag przez model i różni się między pomiarami.

Pomiar ogólny systemu po restarcie (`free -h`):

```text
               total        used        free      shared  buff/cache   available
Mem:           5.3Gi       1.7Gi        72Mi       5.2Mi       3.7Gi       3.5Gi
Swap:          3.8Gi       251Mi       3.5Gi
```

Fragment `top -b -n1`:

```text
MiB Mem :   5384.5 total,     72.6 free,   1766.8 used,   3829.2 buff/cache
MiB Swap:   3842.0 total,   3590.6 free,    251.4 used.   3617.7 avail Mem
```

Z pomiarów wynika, że po załadowaniu modelu proces zajmuje ok. 1,7-1,8 GiB pamięci (`used`), a pozostałe ok. 3,7 GiB to cache pliku modelu (`buff/cache`), który system może w razie potrzeby zwolnić. Dostępne pozostaje ok. 3,5 GiB. Maszyna ma ok. 5,3 GiB RAM, więc model 4B mieści się z zapasem. W razie problemów z pamięcią można ograniczyć kontekst, dopisując do `ExecStart` flagi `-c 4096 --parallel 1`.

### Automatyczny restart po `kill`

Proces główny o PID 3199 został zabity:

```bash
sudo kill $(systemctl show -p MainPID --value llamafile)
```

Stan przed: `MainPID przed kill: 3199`. Stan po 20 sekundach (systemd uruchomił usługę sam, nowy PID):

```text
● llamafile.service - Llamafile LLM server
     Loaded: loaded (/etc/systemd/system/llamafile.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-10-26 17:07:23 UTC; 18s ago
   Main PID: 3285 (.ape-1.10)
      Tasks: 14 (limit: 6497)
     Memory: 1.8G (peak: 1.8G, swap: 67.6M, swap peak: 216.8M)
     CGroup: /system.slice/llamafile.service
             └─3285 /var/lib/llamafile/.ape-1.10 /opt/llamafile/model.llamafile --server --gpu disable --host 192.168.128.2 --port 8080
...
Oct 26 17:07:29 kaszmirowymnich sh[3285]: ... llama_server: listening on http://192.168.128.2:8080
```

PID zmienił się z 3199 na 3285, a usługa jest `active (running)`.

Uwaga: proces nazywa się `.ape-1.10`, więc PID najlepiej pobierać przez `systemctl show -p MainPID llamafile`, a nie przez `ps aux | grep llamafile`.

### Autostart po restarcie maszyny

```bash
sudo reboot
# po ponownym zalogowaniu:
systemctl status llamafile --no-pager
```

```text
● llamafile.service - Llamafile LLM server
     Loaded: loaded (/etc/systemd/system/llamafile.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-10-26 23:48:28 UTC; 1min 20s ago
   Main PID: 1047 (.ape-1.10)
      Tasks: 14 (limit: 6497)
     Memory: 4.6G (peak: 4.7G, swap: 243.6M, swap peak: 244.9M)
     CGroup: /system.slice/llamafile.service
             └─1047 /var/lib/llamafile/.ape-1.10 /opt/llamafile/model.llamafile --server --gpu disable --host 192.168.128.2 --port 8080
...
Oct 26 23:48:35 kaszmirowymnich sh[1047]: ... llama_server: listening on http://192.168.128.2:8080
```

Usługa wystartowała sama po restarcie systemu, bez ręcznej interwencji.

## Najczęstsze problemy

| Objaw | Przyczyna i rozwiązanie |
|---|---|
| `mkdir: cannot create directory '/home/llamafile'` | Brak katalogu domowego. Utwórz `/var/lib/llamafile` i ustaw `HOME` (kroki 1 i 5). |
| `curl: (23) Failure writing output` | Brak miejsca w `/tmp`. Pobieraj bezpośrednio do `/opt/llamafile`. |
| `status=203/EXEC` | Dodaj `/bin/sh` przed ścieżką do pliku w `ExecStart`. |
| `curl: (28)` timeout z gospodarza | Zapora `ufw` blokuje port 8080 (krok 4). |
| `Cannot assign requested address` | Adres host-only nie jest przypisany. Sprawdź `ip -br a` i konfigurację netplan. |
| `nano`: `cannot initialize terminal type xterm-ghostty` | Użytkownicy Ghostty: `export TERM=xterm-256color` przed edycją. |

## Struktura repozytorium

```text
.
├── README.md
├── .gitignore
├── config/
│   └── llamafile.service
└── outputs/
    ├── ss.txt
    ├── status.txt
    ├── top.txt
    ├── free.txt
    ├── kill-przed.txt
    ├── kill-po.txt
    └── reboot.txt
```

Plik modelu (`*.llamafile`, ok. 3,9 GB) nie jest dodawany do repozytorium.
