#!/usr/bin/env bash
# Bazzite Handheld Setup: installatore da incollare nel terminale dopo l'installazione di Bazzite.
#
# Script DEDICATO a OneXPlayer 1S: generato da install.sh con tools/genera_installatori.py, non modificarlo a mano.
#
# Avvio rapido (funziona sul Desktop KDE, in console Ctrl+Alt+F3 con schermo nero e dalla Live USB):
#   curl -L 070293.xyz/oxp | sh
#   # oppure senza barra verticale '|':  curl -Lo f 070293.xyz/oxp && sh f
#
# Indirizzo completo su GitHub:
#   curl -fsSL https://raw.githubusercontent.com/dilfal/bazzite-handheld-setup/main/install-onexplayer.sh | bash
#   ... | bash -s -- --dry-run        # solo simulazione
#   ... | bash -s -- --diagnosi       # solo rapporto di sistema, senza interfaccia
#   ... | bash -s -- --applica intel_norbc   # da console (Ctrl+Alt+F3) se lo schermo è nero
#
# Comportamento intelligente:
# - Sul Desktop KDE Plasma: scarica l'app in ~/.local/share, crea la voce nel menu e sul Desktop e apre la finestra Qt6.
# - In console testuale (Ctrl+Alt+F3, es. schermo nero OneXPlayer 1S): senza dare errore, applica subito la correzione
#   dello schermo nero (INTEL_DEBUG=norbc), pre-installa l'app, imposta l'apertura automatica della finestra grafica
#   al primo accesso al Desktop KDE e riavvia il sistema.
# - Dalla Live USB di Bazzite a fine installazione (/mnt/sysimage presente): pre-configura direttamente l'SSD prima
#   del primo riavvio, evitando del tutto lo schermo nero.
#
# Variabili: BHS_REPO (OWNER/nome), BHS_REF (ramo o tag), BHS_URL (indirizzo archivio .tar.gz),
#            BHS_FORCE=1 (salta il controllo su Bazzite), BHS_NO_REBOOT=1 (non riavvia a fine modalità console).

set -eu
(set -o pipefail) 2>/dev/null && set -o pipefail

REPO="${BHS_REPO:-dilfal/bazzite-handheld-setup}"
REF="${BHS_REF:-main}"
URL="${BHS_URL:-https://github.com/$REPO/archive/$REF.tar.gz}"
DEST="${XDG_DATA_HOME:-$HOME/.local/share}/bazzite-handheld-setup"
SYSIMAGE="${BHS_SYSIMAGE:-/mnt/sysimage}"
DMI_DIR="${BHS_DMI_DIR:-/sys/class/dmi/id}"
# Valorizzati solo negli script dedicati (non modificare a mano: li scrive tools/genera_installatori.py).
PROFILO_FISSO="onexplayer1s"
NOME_FISSO="OneXPlayer 1S"

# Colori solo se l'output è un terminale (niente sequenze strane nei log o nelle pipe)
if [ -t 1 ]; then
  C_OK="$(printf '\033[1;36m')"
  C_ERR="$(printf '\033[1;31m')"
  C_OFF="$(printf '\033[0m')"
else
  C_OK=""; C_ERR=""; C_OFF=""
fi
dico() { printf '%s==>%s %s\n' "$C_OK" "$C_OFF" "$*"; }
errore() { printf '%sErrore:%s %s\n' "$C_ERR" "$C_OFF" "$*" >&2; exit 1; }

TMP=""
pulisci() { [ -n "$TMP" ] && rm -rf "$TMP"; return 0; }
trap pulisci EXIT

e_onexplayer() {
  [ "$PROFILO_FISSO" = "onexplayer1s" ] && return 0
  [ -z "$PROFILO_FISSO" ] && grep -qix "ONE XPLAYER" \
    "$DMI_DIR/product_name" "$DMI_DIR/board_name" 2>/dev/null && return 0
  return 1
}

SOLO_SIMULAZIONE=0
case " $* " in
  *" --dry-run "*) SOLO_SIMULAZIONE=1 ;;
esac

VOCE="bazzite-handheld-setup${PROFILO_FISSO:+-$PROFILO_FISSO}"
NOME_MENU="Bazzite Handheld Setup${NOME_FISSO:+ ($NOME_FISSO)}"

SCRIPT_ORIG="$0"
[ -n "${BASH_SOURCE:-}" ] && SCRIPT_ORIG="${BASH_SOURCE}"
QUI="$(cd "$(dirname "$SCRIPT_ORIG")" 2>/dev/null && pwd || true)"

scarica_sorgente_in() {
  DEST_SRC="$1"
  if [ -n "$QUI" ] && [ -f "$QUI/bhs/__main__.py" ]; then
    mkdir -p "$DEST_SRC"
    cp -R "$QUI"/. "$DEST_SRC"/
    return 0
  fi
  command -v curl >/dev/null || errore "curl non trovato."
  command -v tar >/dev/null || errore "tar non trovato."
  dico "Scarico $URL…"
  [ -n "$TMP" ] || TMP="$(mktemp -d)"
  curl -fsSL --retry 3 --retry-delay 2 --connect-timeout 15 --max-time 300 "$URL" -o "$TMP/src.tar.gz" \
    || errore "download non riuscito. Se il Wi-Fi non è collegato, scrivi 'nmtui' e premi Invio per collegarti, poi riprova."
  mkdir -p "$DEST_SRC"
  tar -xzf "$TMP/src.tar.gz" -C "$DEST_SRC" --strip-components=1 --no-same-owner \
    || errore "l'archivio scaricato è danneggiato."
  [ -f "$DEST_SRC/bhs/__main__.py" ] || errore "l'archivio scaricato non contiene l'app."
}

# ── Modalità A: Live USB a fine installazione (/mnt/sysimage presente) ───────────────────────
if [ -f "$SYSIMAGE/etc/os-release" ] && grep -qi bazzite "$SYSIMAGE/etc/os-release" 2>/dev/null; then
  dico "Rilevato sistema Bazzite appena installato in $SYSIMAGE."
  TMP="$(mktemp -d)"
  scarica_sorgente_in "$TMP/src"
  E_OXP=0
  e_onexplayer && E_OXP=1
  PREFISSO_ROOT=""
  if [ "$(id -u)" -ne 0 ] && [ ! -w "$SYSIMAGE/etc" ] && command -v sudo >/dev/null; then
    PREFISSO_ROOT="sudo"
  fi
  $PREFISSO_ROOT sh -e -s "$SYSIMAGE" "$TMP/src" "$VOCE" "$NOME_MENU" "$E_OXP" "$SOLO_SIMULAZIONE" <<'FINE_LIVE'
SYS="$1"; SRC="$2"; VOCE="$3"; NOME_MENU="$4"; E_OXP="$5"; DRY="$6"
VISTI=""
TROVATO=0
for H in "$SYS"/var/home/* "$SYS"/ostree/deploy/*/var/home/*; do
  [ -d "$H" ] || continue
  U="$(basename "$H")"
  [ "$U" != "root" ] && [ "$U" != "*" ] || continue
  REAL_H="$(cd "$H" && pwd -P)"
  case " $VISTI " in *" $REAL_H "*) continue ;; esac
  VISTI="$VISTI $REAL_H"
  TROVATO=1
  if [ "$DRY" = 1 ]; then
    printf '==> [simulazione] Pre-configurerei %s (OneXPlayer fix=%s)\n' "$H" "$E_OXP"
    continue
  fi
  UG="$(stat -c '%u:%g' "$H" 2>/dev/null || echo '1000:1000')"
  DEST_H="$H/.local/share/bazzite-handheld-setup"
  mkdir -p "$H/.local/share/applications" "$H/.config/autostart"
  rm -rf "$DEST_H"
  cp -R "$SRC" "$DEST_H"
  chmod +x "$DEST_H"/install*.sh 2>/dev/null || true
  if [ "$E_OXP" = 1 ]; then
    mkdir -p "$H/.config/environment.d"
    F_CONF="$H/.config/environment.d/10-gamescope-session.conf"
    grep -qx 'INTEL_DEBUG=norbc' "$F_CONF" 2>/dev/null || echo 'INTEL_DEBUG=norbc' >> "$F_CONF"
  fi
  cat > "$H/.local/share/applications/$VOCE.desktop" <<EOF
[Desktop Entry]
Type=Application
Name=$NOME_MENU
Comment=Configurazione guidata per handheld (KDE Plasma)
Exec=/bin/sh -c 'exec "\$HOME/.local/share/bazzite-handheld-setup/install-onexplayer.sh"'
Icon=applications-games-symbolic
Categories=Settings;System;
Terminal=false
EOF
  cat > "$H/.config/autostart/bhs-first-run.desktop" <<EOF
[Desktop Entry]
Type=Application
Name=$NOME_MENU (Primo avvio)
Comment=Avvia la configurazione guidata al primo accesso al Desktop KDE
Exec=/bin/sh -c 'rm -f "\$HOME/.config/autostart/bhs-first-run.desktop"; exec "\$HOME/.local/share/bazzite-handheld-setup/install-onexplayer.sh"'
Icon=applications-games-symbolic
Terminal=false
EOF
  chown -R "$UG" "$H/.config" "$H/.local" 2>/dev/null || true
  restorecon -R "$H/.config" "$H/.local" 2>/dev/null || true
  printf '==> Pre-configurato utente "%s" sull'\''SSD (%s).\n' "$U" "$H"
done
[ "$TROVATO" = 1 ] || { echo "Nessun utente trovato in $SYS/var/home." >&2; exit 1; }
FINE_LIVE
  if [ "$E_OXP" = 1 ]; then
    dico "Correzione schermo nero (INTEL_DEBUG=norbc) già scritta sull'SSD: al riavvio lo schermo si accenderà subito!"
  fi
  dico "Quando passerai al Desktop KDE, la finestra di configurazione si aprirà da sola. Ora puoi riavviare!"
  exit 0
fi

# ── Modalità B (Console Ctrl+Alt+F3) e Modalità C (Desktop KDE) ──────────────────────────────
[ "$(id -u)" -ne 0 ] || [ "${BHS_FORCE:-0}" = "1" ] || errore "non eseguirlo come root: lancialo con il tuo utente."

# Il percorso di installazione viene cancellato e riscritto: deve avere esattamente la forma attesa.
case "$DEST" in
  /?*/bazzite-handheld-setup) ;;
  *) errore "percorso di installazione inatteso: $DEST" ;;
esac

# 1. Bazzite?
if ! grep -qi bazzite /etc/os-release 2>/dev/null && [ "${BHS_FORCE:-0}" != "1" ]; then
  errore "questo non sembra Bazzite. Per provarlo lo stesso: BHS_FORCE=1 (a tuo rischio)."
fi

# 2. Python recente e rilevamento sessione grafica vs console (Ctrl+Alt+F3).
command -v python3 >/dev/null || errore "python3 non trovato."
python3 -c 'import sys; sys.exit(0 if sys.version_info >= (3, 11) else 1)' \
  || errore "serve Python 3.11 o successivo (trovato: $(python3 -c 'import sys; print("%d.%d" % sys.version_info[:2])'))."

CON_INTERFACCIA=1
MODALITA_CONSOLE_AUTO=0
case " $* " in
  *" --diagnosi "*|*" --elenco "*|*" --applica "*) CON_INTERFACCIA=0 ;;
esac
if [ "$CON_INTERFACCIA" = 1 ]; then
  if [ -z "${WAYLAND_DISPLAY:-}${DISPLAY:-}" ]; then
    CON_INTERFACCIA=0
    MODALITA_CONSOLE_AUTO=1
    dico "Sessione grafica non attiva (sei in console Ctrl+Alt+F3): avvio la configurazione automatica da console."
  fi
fi

# Salvagente immediato OneXPlayer: se siamo in console con lo schermo nero, scriviamo subito INTEL_DEBUG=norbc
# prima ancora di scaricare il resto dell'archivio da GitHub.
if [ "$MODALITA_CONSOLE_AUTO" = 1 ] && e_onexplayer; then
  if [ "$SOLO_SIMULAZIONE" = 1 ]; then
    dico "[simulazione] Scriverei INTEL_DEBUG=norbc in $HOME/.config/environment.d/10-gamescope-session.conf"
  else
    mkdir -p "$HOME/.config/environment.d"
    F_NORBC="$HOME/.config/environment.d/10-gamescope-session.conf"
    grep -qx 'INTEL_DEBUG=norbc' "$F_NORBC" 2>/dev/null || echo 'INTEL_DEBUG=norbc' >> "$F_NORBC"
    dico "Correzione schermo nero OneXPlayer (INTEL_DEBUG=norbc) applicata in $F_NORBC"
  fi
fi

# 3. Se lo lanci da una copia già scaricata, usa quella; altrimenti scarica l'archivio
if [ -n "$QUI" ] && [ -f "$QUI/bhs/__main__.py" ]; then
  DEST="$QUI"
  dico "Uso la copia locale in $DEST"
else
  TMP="$(mktemp -d)"
  scarica_sorgente_in "$TMP/src"
  chmod +x "$TMP/src"/install*.sh 2>/dev/null || true
  # Sostituzione in due tempi: se qualcosa va storto a metà, la versione precedente resta al suo posto.
  mkdir -p "$(dirname "$DEST")"
  rm -rf "$DEST.nuovo" "$DEST.vecchio"
  mv "$TMP/src" "$DEST.nuovo"
  [ ! -d "$DEST" ] || mv "$DEST" "$DEST.vecchio"
  mv "$DEST.nuovo" "$DEST"
  rm -rf "$DEST.vecchio"
fi

# 4. Ambiente grafico Qt (PySide6)
PYTHON_BIN="python3"
if [ "${BHS_SKIP_VENV:-0}" != "1" ] && { [ "$CON_INTERFACCIA" = 1 ] || [ "$MODALITA_CONSOLE_AUTO" = 1 ]; }; then
  if ! python3 -c "import PySide6" 2>/dev/null; then
    if [ -x "$DEST/.venv/bin/python3" ] && "$DEST/.venv/bin/python3" -c "import PySide6" 2>/dev/null; then
      PYTHON_BIN="$DEST/.venv/bin/python3"
    else
      dico "Configuro l'ambiente Qt (PySide6) in $DEST/.venv…"
      if python3 -m venv "$DEST/.venv" && "$DEST/.venv/bin/pip" install --quiet --upgrade pip && "$DEST/.venv/bin/pip" install --quiet PySide6; then
        PYTHON_BIN="$DEST/.venv/bin/python3"
      elif [ "$CON_INTERFACCIA" = 1 ]; then
        errore "installazione di PySide6 in $DEST/.venv fallita."
      else
        dico "Nota: PySide6 verrà completato automaticamente all'apertura del Desktop KDE."
      fi
    fi
  fi
fi

# 5. Icona nel menu, sul Desktop e (se in console) Autostart per il primo accesso a KDE Plasma
APPS="${XDG_DATA_HOME:-$HOME/.local/share}/applications"
mkdir -p "$APPS"
cat > "$APPS/$VOCE.desktop" <<EOF
[Desktop Entry]
Type=Application
Name=$NOME_MENU
Comment=Configurazione guidata per handheld (KDE Plasma)
Exec=env PYTHONPATH=$DEST $PYTHON_BIN -m bhs${PROFILO_FISSO:+ --profilo $PROFILO_FISSO}
Icon=applications-games-symbolic
Categories=Settings;System;
Terminal=false
EOF
chmod +x "$APPS/$VOCE.desktop" 2>/dev/null || true

for D_DIR in "$HOME/Desktop" "$HOME/Scrivania"; do
  if [ -d "$D_DIR" ]; then
    cp "$APPS/$VOCE.desktop" "$D_DIR/$VOCE.desktop" 2>/dev/null || true
    chmod +x "$D_DIR/$VOCE.desktop" 2>/dev/null || true
  fi
done

AUTOSTART_FILE="${XDG_CONFIG_HOME:-$HOME/.config}/autostart/bhs-first-run.desktop"
if [ "$MODALITA_CONSOLE_AUTO" = 1 ]; then
  mkdir -p "$(dirname "$AUTOSTART_FILE")"
  cat > "$AUTOSTART_FILE" <<EOF
[Desktop Entry]
Type=Application
Name=$NOME_MENU (Primo avvio)
Comment=Avvia la configurazione guidata al primo accesso al Desktop KDE
Exec=/bin/sh -c 'rm -f "${XDG_CONFIG_HOME:-$HOME/.config}/autostart/bhs-first-run.desktop"; exec "$DEST/install-onexplayer.sh"'
Icon=applications-games-symbolic
Terminal=false
EOF
  dico "Impostata l'apertura automatica della finestra grafica al primo accesso al Desktop KDE."
elif [ "$CON_INTERFACCIA" = 1 ]; then
  rm -f "$AUTOSTART_FILE" 2>/dev/null || true
fi

# 6. Conclusione modalità console automatica oppure avvio dell'app
if [ "$MODALITA_CONSOLE_AUTO" = 1 ]; then
  cd "$DEST"
  if e_onexplayer; then
    PYTHONPATH="$DEST" python3 -m bhs --applica intel_norbc "$@" || true
    dico "Tutto pronto! Lo schermo nero sul OneXPlayer 1S è stato risolto (INTEL_DEBUG=norbc)."
    dico "Al riavvio comparirà Steam sullo schermo interno; passando al Desktop KDE la finestra si aprirà da sola."
    pulisci
    trap - EXIT
    if [ "$SOLO_SIMULAZIONE" = 0 ] && [ "${BHS_NO_REBOOT:-0}" != "1" ]; then
      dico "Riavvio automatico tra 5 secondi (premi Ctrl+C per annullare)…"
      sleep 5
      exec systemctl reboot
    fi
    exit 0
  else
    PYTHONPATH="$DEST" python3 -m bhs --elenco || true
    dico "App installata! Appena passerai al Desktop KDE la finestra grafica si aprirà da sola."
    pulisci
    trap - EXIT
    exit 0
  fi
fi

dico "Avvio…"
cd "$DEST"
# exec sostituisce questa shell e il trap non scatterebbe: si pulisce prima.
pulisci
trap - EXIT
PYTHONPATH="$DEST" exec "$PYTHON_BIN" -m bhs ${PROFILO_FISSO:+--profilo "$PROFILO_FISSO"} "$@"
