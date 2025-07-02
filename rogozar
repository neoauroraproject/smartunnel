#!/bin/bash

# Rogozar Tunnel Setup Script
# Version: v1.0
# @coderman_ir

# Colors
GREEN="\e[32m"
RED="\e[31m"
YELLOW="\e[33m"
RESET="\e[0m"

function pause() {
  echo
  read -rp "Press ENTER to continue..."
}

function show_header() {
  clear
  cat << "EOF"
______
| ___ \
| |_/ /___   __ _  ___ ______ _ _ __
|    // _ \ / _` |/ _ \_  / _` | '__|
| |\ \ (_) | (_| | (_) / / (_| | |
\_| \_\___/ \__, |\___/___\__,_|_|
             __/ |
            |___/
EOF
  echo -e "${GREEN}Welcome to Rogozar Tunnel Setup Script${RESET}"
  echo -e "Version: v1.0"
  echo -e "@coderman_ir"
  echo
  # Show IP Server:
  IP=$(hostname -I 2>/dev/null | awk '{print $1}')
  echo -e "IP Server: ${YELLOW}${IP}${RESET}"

  # Check gost core installed or not:
  if command -v gost >/dev/null 2>&1; then
    echo -e "Core: ${GREEN}installed${RESET}"
  else
    echo -e "Core: ${RED}uninstalled${RESET}"
  fi
  echo
}

function show_menu() {
  show_header
  echo "1) Setup / Configure Tunnel"
  echo "2) Edit Existing Tunnel"
  echo "3) Remove Tunnel"
  echo "4) Restart Tunnel"
  echo "5) Install gost Core"
  echo "6) Exit"
  echo -n "Your choice: "
  read -r choice
}

function install_gost_core() {
  echo -e "${GREEN}Installing gost core...${RESET}"
  bash <(curl -fsSL https://github.com/go-gost/gost/raw/master/install.sh) --install

  if command -v gost >/dev/null 2>&1; then
    echo -e "${GREEN}gost core installed successfully.${RESET}"
  else
    echo -e "${RED}Failed to install gost core.${RESET}"
  fi
  pause
}

function setup_tunnel() {
  echo -n "Enter tunnel name (e.g. iran01): "
  read -r NAME
  SERVICE_NAME="rogozar-$NAME"

  # Check if service already exists
  if systemctl list-units --all --type=service | grep -q "^$SERVICE_NAME.service"; then
    echo -e "${RED}Service '$SERVICE_NAME' already exists! Choose another name.${RESET}"
    pause
    return
  fi

  echo -n "Is this server located in Iran or Kharej? (1 = Iran / 2 = Kharej): "
  read -r location

  echo -n "Connection Username: "
  read -r username
  echo -n "Connection Password: "
  read -rs password
  echo
  echo -n "WSS Port (default 8443): "
  read -r port
  port=${port:-8443}

  if [ "$location" = "1" ]; then
    echo "Setting up Iranian-side server (bind=true)..."
    CMD="relay+wss://$username:$password@:$port?bind=true"
    EXEC_CMD="/usr/local/bin/gost -L $CMD"
  else
    echo -n "Domain of Iranian server (e.g. iran.example.com): "
    read -r iran_domain
    echo -n "Tunnel target ports (comma-separated, e.g. 80,443,8080): "
    read -r ports

    PORT_FORWARD=""
    IFS=',' read -ra PORTS <<< "$ports"
    for p in "${PORTS[@]}"; do
      PORT_FORWARD+=" -L rtcp://:$p/:$p"
    done

    CMD="$PORT_FORWARD -F relay+wss://$username:$password@$iran_domain:$port"
    EXEC_CMD="/usr/local/bin/gost $CMD"
  fi

  echo "Creating systemd service..."

  cat <<EOF | sudo tee /etc/systemd/system/$SERVICE_NAME.service > /dev/null
[Unit]
Description=Rogozar Tunnel ($SERVICE_NAME)
After=network.target

[Service]
ExecStart=$EXEC_CMD
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

  sudo systemctl daemon-reexec
  sudo systemctl daemon-reload
  sudo systemctl enable --now $SERVICE_NAME

  echo -e "${GREEN}Tunnel '$SERVICE_NAME' created and enabled on boot.${RESET}"
  pause
}

function select_tunnel() {
  mapfile -t services < <(ls /etc/systemd/system/ | grep "^rogozar-.*\.service" | sed 's/\.service$//')

  if [ ${#services[@]} -eq 0 ]; then
    echo -e "${RED}No rogozar-* tunnels found.${RESET}"
    selected_service=""
    return 1
  fi

  echo "Available tunnels:"
  for i in "${!services[@]}"; do
    echo "$((i+1))) ${services[$i]}"
  done

  echo -n "Select tunnel (1-${#services[@]}): "
  read -r index

  if ! [[ "$index" =~ ^[0-9]+$ ]] || [ "$index" -lt 1 ] || [ "$index" -gt "${#services[@]}" ]; then
    echo -e "${RED}Invalid selection.${RESET}"
    selected_service=""
    return 1
  fi

  selected_service="${services[$((index-1))]}"
  return 0
}

function remove_tunnel() {
  if ! select_tunnel; then
    pause
    return
  fi

  sudo systemctl stop "$selected_service"
  sudo systemctl disable "$selected_service"
  sudo rm -f "/etc/systemd/system/$selected_service.service"
  sudo systemctl daemon-reload

  echo -e "${RED}Tunnel '$selected_service' has been removed.${RESET}"
  pause
}

function edit_tunnel() {
  if ! select_tunnel; then
    pause
    return
  fi

  echo -e "${GREEN}Opening systemd config for editing...${RESET}"
  sudo nano "/etc/systemd/system/$selected_service.service"
  sudo systemctl daemon-reload
  sudo systemctl restart "$selected_service"

  echo -e "${GREEN}Tunnel '$selected_service' updated and restarted.${RESET}"
  pause
}

function restart_tunnel() {
  if ! select_tunnel; then
    pause
    return
  fi

  echo -e "${YELLOW}Restarting tunnel '$selected_service'...${RESET}"
  sudo systemctl restart "$selected_service"

  # Check status after restart
  sleep 2
  if systemctl is-active --quiet "$selected_service"; then
    echo -e "${GREEN}Tunnel '$selected_service' restarted successfully and is active.${RESET}"
  else
    echo -e "${RED}Tunnel '$selected_service' failed to restart or is inactive.${RESET}"
  fi
  pause
}

while true; do
  show_menu
  case $choice in
    1) setup_tunnel ;;
    2) edit_tunnel ;;
    3) remove_tunnel ;;
    4) restart_tunnel ;;
    5) install_gost_core ;;
    6) exit 0 ;;
    *) echo "Invalid choice. Please enter 1–6." ; pause ;;
  esac
done
