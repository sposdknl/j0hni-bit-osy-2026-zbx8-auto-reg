
IMAGE_NAME = "bento/debian-13"
ZABBIX_HOSTNAME = "Andreovsky_Jan_Debian13"
ZABBIX_SERVER = "enceladus.pfsense.cz"

Vagrant.configure("2") do |config|

    config.ssh.insert_key = true

    # ==========================================
    # VIRTUALBOX
    # ==========================================

    config.vm.provider "virtualbox" do |v|
        v.memory = 4096
        v.cpus = 4
    end

    # ==========================================
    # DEBIAN 13
    # ==========================================

    config.vm.define "debian" do |debian|

        debian.vm.box = IMAGE_NAME
        debian.vm.hostname = "debian13"

        # SSH access
        debian.vm.network "forwarded_port",
            guest: 22,
            host: 2206,
            host_ip: "127.0.0.1"

        # Apache HTTP
        debian.vm.network "forwarded_port",
            guest: 80,
            host: 8080,
            host_ip: "127.0.0.1"

        # ==========================================
        # PROVISIONING
        # ==========================================

        debian.vm.provision "shell",
            env: {
                "ORIGINAL_HOSTNAME" => ZABBIX_HOSTNAME,
                "ZABBIX_SERVER" => ZABBIX_SERVER
            },
            inline: <<-SHELL

            set -e

            export DEBIAN_FRONTEND=noninteractive

            echo "======================================"
            echo " Aktualizace systemu"
            echo "======================================"

            apt-get update
            apt-get install -y apache2 curl ca-certificates

            # ==========================================
            # ZABBIX 8.0 OFFICIAL REPOSITORY
            # ==========================================

            echo "Instalace oficialniho Zabbix 8.0 repozitare..."

            cd /tmp

            curl -fLO https://repo.zabbix.com/zabbix/8.0/release/debian/pool/main/z/zabbix-release/zabbix-release_latest+debian13_all.deb

            dpkg -i zabbix-release_latest+debian13_all.deb

            apt-get update

            # ==========================================
            # ZABBIX AGENT 2
            # ==========================================

            echo "Instalace Zabbix Agent 2..."

            apt-get install -y zabbix-agent2

            # Check version
            AGENT_VERSION=$(zabbix_agent2 -V 2>&1 | head -n 1)

            echo "Nainstalovana verze: $AGENT_VERSION"

            case "$AGENT_VERSION" in
                *"8.0."*)
                    echo "Zabbix Agent 2 verze 8.0 potvrzena."
                    ;;
                *)
                    echo "CHYBA: Nebyla nainstalovana verze 8.0!"
                    exit 1
                    ;;
            esac

            # ==========================================
            # APACHE - MOD_STATUS
            # ==========================================

            echo "Konfigurace Apache..."

            a2enmod status

            cat > /etc/apache2/conf-available/server-status.conf <<'APACHECONF'
<Location "/server-status">
    SetHandler server-status
    Require all granted
</Location>
APACHECONF

            a2enconf server-status

            apache2ctl configtest

            systemctl enable apache2
            systemctl restart apache2

            if ! systemctl is-active --quiet apache2; then
                echo "CHYBA: Apache nebezi!"
                journalctl -u apache2 -n 50 --no-pager || true
                exit 1
            fi

            # ==========================================
            # ZABBIX AGENT 2 CONFIGURATION
            # ==========================================

            echo "Konfigurace Zabbix Agent 2..."

            CONFIG_FILE="/etc/zabbix/zabbix_agent2.conf"

            # Remove previous active directives
            sed -i -E \
                '/^[[:space:]]*(Server|ServerActive|Hostname|HostMetadata)=/d' \
                "$CONFIG_FILE"

            # Write required configuration
            cat >> "$CONFIG_FILE" <<EOF

# ======================================
# Vagrant - Zabbix Agent 2
# ======================================

Server=${ZABBIX_SERVER}
ServerActive=${ZABBIX_SERVER}
Hostname=${ORIGINAL_HOSTNAME}
HostMetadata=SPOS

EOF

            # ==========================================
            # START ZABBIX AGENT 2
            # ==========================================

            echo "Spousteni Zabbix Agent 2..."

            systemctl enable zabbix-agent2
            systemctl restart zabbix-agent2

            # Check configuration
            zabbix_agent2 -t agent.ping

            if ! systemctl is-active --quiet zabbix-agent2; then
                echo "CHYBA: Zabbix Agent 2 nebezi!"
                journalctl -u zabbix-agent2 -n 50 --no-pager || true
                exit 1
            fi

            echo "Zabbix Agent 2 je spusteny."

            # ==========================================
            # FINAL CHECK
            # ==========================================

            echo "======================================"
            echo " Kontrola sluzeb"
            echo "======================================"

            systemctl is-active apache2
            systemctl is-active zabbix-agent2

            echo "======================================"
            echo " Provisioning dokoncen uspesne!"
            echo "======================================"

            echo "Hostname: ${ORIGINAL_HOSTNAME}"
            echo "Zabbix server: ${ZABBIX_SERVER}"
            echo "Metadata: SPOS"

            echo "Apache: http://localhost:8080"
            echo "Apache status: http://localhost:8080/server-status"
            echo "SSH: vagrant ssh"

            SHELL

    end
end
