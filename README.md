# AMD BC-250 — Headless Streaming & Gaming Setup

Setup completo para usar a BC-250 como PC de gaming headless via Moonlight/Sunshine no CachyOS.

## Hardware

- **CPU/GPU**: AMD BC-250 (Cyan Skillfish, GFX1013, RDNA 1.5, 6C/12T Zen 2)
- **RAM**: 16 GB GDDR6 (compartilhada CPU/GPU)
- **VRAM**: 512 MB dedicada + GTT dinâmico (até 14 GB)
- **Disco**: NVMe 477 GB
- **OS**: CachyOS Linux (Arch-based)
- **VCN**: Desabilitado — firmware bloqueado pela Sony. Sem hardware encode/decode. CPU only

---

## 1. Instalar CachyOS

Instalar normalmente pelo ISO padrão.

### Compatibilidade de kernels

| Kernel | Status | Notas |
|--------|--------|-------|
| 6.10.x | ⚠️ Funciona | `amdgpu.sg_display=0` obrigatório |
| 6.11.x–6.14.x LTS | ✅ Bom | Estável |
| **6.15.0–6.15.6** | ❌ **Quebrado** | GPU init falha / kernel panic |
| 6.15.7–6.17.7 | ✅ **Recomendado** | Melhor suporte |
| **6.17.8–6.17.10** | ❌ **Quebrado** | Driver GPU quebrado |
| 6.17.11+ | ✅ **Recomendado** | Fix aplicado |

```bash
uname -r  # verificar versão atual
```

> ⚠️ **NUNCA use `amd_iommu=on`** — IOMMU está quebrado na BC-250 e causa travamentos e falhas de display.

### Pacotes essenciais

```bash
sudo pacman -S mesa vulkan-radeon lib32-vulkan-radeon steam mangohud gamemode \
  sway wlr-randr labwc foot fuzzel pipewire pipewire-pulse wireplumber \
  xdg-desktop-portal-wlr seatd nvtop htop lm_sensors
```

---

## 2. GPU Governor

Sem governor, a GPU fica travada em 1500 MHz.

O CachyOS já pode ter o `cyan-skillfish-governor-smu` disponível:

```bash
sudo pacman -S cyan-skillfish-governor-smu  # ou instalar via AUR
sudo systemctl enable --now cyan-skillfish-governor-smu
```

> **Nota CachyOS**: o governor pode não iniciar no boot mesmo habilitado. Comportamento normal — funciona corretamente após abrir qualquer jogo pela primeira vez.

### Configuração (`/etc/cyan-skillfish-governor-smu/config.toml`)

```toml
[timing.intervals]
sample = 500
adjust = 200_000

[timing.ramp-rates]
normal = 1
burst = 50

[timing]
burst-samples = 60
down-events = 5

[frequency-thresholds]
adjust = 10

[load-target]
upper = 0.80
lower = 0.65

[temperature]
throttling = 95
throttling_recovery = 85

[[safe-points]]
frequency = 1000
voltage = 800

[[safe-points]]
frequency = 1175
voltage = 850

[[safe-points]]
frequency = 1500
voltage = 900

[[safe-points]]
frequency = 1600
voltage = 910

[[safe-points]]
frequency = 1700
voltage = 920

[[safe-points]]
frequency = 1850
voltage = 930

[[safe-points]]
frequency = 2000
voltage = 960

[[safe-points]]
frequency = 2050
voltage = 980

[[safe-points]]
frequency = 2100
voltage = 1000

[[safe-points]]
frequency = 2150
voltage = 1015

[[safe-points]]
frequency = 2200
voltage = 1030
```

Verificar:
```bash
cat /sys/class/drm/card0/device/pp_dpm_sclk
# Deve mostrar múltiplas frequências, * indica a ativa
```

---

## 3. VRAM e Memória

### BIOS

Configurar UMA Frame Buffer Size para **512 MB** (modo dinâmico). A GPU aloca mais conforme necessário.

### Expandir GTT para 14 GB

> **Por que 14 GB?** Com o ajuste correto de memória do sistema (`no_system_mem_limit=1`), 14 GB permite rodar modelos de IA maiores (SDXL, LLMs) sem swapping excessivo. Se notar crashes constantes em tarefas compute, reduza para 12 GB.

```bash
# /etc/modprobe.d/amdgpu-mem.conf
options amdgpu no_system_mem_limit=1 gttsize=14336
```

### Swap no NVMe (Obrigatório para GTT > 12GB)

Como o GTT de 14 GB usa quase toda a RAM física (16 GB), é **essencial** ter um swap file grande no NVMe para evitar que o sistema trave (OOM) em picos de uso. Recomendado: **16 GB a 32 GB**.

```bash
sudo fallocate -l 32G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo "/swapfile none swap defaults,pri=10 0 0" | sudo tee -a /etc/fstab
```

Rebuild initramfs:
```bash
sudo mkinitcpio -P
```

Verificar após reboot:
```bash
vulkaninfo 2>&1 | grep -A3 "memoryHeaps\[0\]"
# Deve mostrar ~14.5 GB
```

---

## 4. Tuning de Sistema e Performance

### Power Profile (Performance)

```bash
powerprofilesctl set performance
```

### Sysctl e Memória (`/etc/sysctl.d/99-bc250-tuning.conf`)

Otimizações para reduzir latência e melhorar o uso da RAM compartilhada:

```ini
vm.swappiness=60
vm.vfs_cache_pressure=50
vm.dirty_ratio=20
vm.dirty_background_ratio=10
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
```

### Audio Latency (PipeWire)

Reduz o atraso do áudio no stream:
```bash
# /etc/environment
PIPEWIRE_QUANTUM=1024/48000
```

### Transparent Huge Pages (THP)

Melhora performance em PyTorch/ROCm:
```bash
echo "madvise" | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo "1" | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

---

## 4.1 Tuning Avançado (Experimental — Estabilidade e Latência)

Configurações de baixo nível para reduzir micro-stutters e prevenir `ring timeout` no driver AMD.

### Kernel Parameters (`/boot/refind_linux.conf` ou GRUB)

```
processor.max_cstate=1 split_lock_detect=off pcie_aspm=off transparent_hugepage=madvise
```

> `processor.max_cstate=1` e `pcie_aspm=off` mantêm o CPU e o barramento PCIe ativos, reduzindo drasticamente a latência de "wake-up".

### Driver AMD Advanced (`/etc/modprobe.d/amdgpu-advanced.conf`)

```bash
options amdgpu sched_hw_submission=2 gpu_recovery=1 audio=0
```

> `sched_hw_submission=2` permite que o hardware gerencie mais comandos simultaneamente, essencial para evitar crashes durante carregamento de modelos de IA. `gpu_recovery=1` tenta resetar apenas o driver se a GPU travar, evitando reboot forçado.

### Sysctl Adicional (`/etc/sysctl.d/99-bc250-tuning.conf`)

```ini
kernel.sched_autogroup_enabled=0
vm.stat_interval=10
net.ipv4.tcp_fastopen=3
net.core.netdev_max_backlog=5000
```

> `net.core.netdev_max_backlog=5000` é vital para streamings de alto bitrate (4K/120Hz) sem perda de pacotes.

### NVMe I/O MQ (`/etc/modprobe.d/scsi-mq.conf`)

```bash
options scsi_mod use_blk_mq=1
```

---

## 5. RADV e Performance

### Navi10 Spoof (fix de detecção de GPU) — Opcional

A BC-250 (GFX1013 / Cyan Skillfish) não é reconhecida por muitos jogos e pelo DXVK. O spoof faz ela se identificar como Navi10 (RX 5700 XT).

```bash
sudo mkdir -p /etc/drirc.d
sudo cp config/99-bc250-navi10.conf /etc/drirc.d/99-bc250-navi10.conf
```

Verificar após reboot:
```bash
vulkaninfo | grep deviceID   # esperado: 0x731F (Navi10)
glxinfo | grep -i device
```

### Variáveis de ambiente (`/etc/environment`)

```
AMD_VULKAN_ICD=RADV
RADV_DEBUG=nohiz,nocompute
MESA_LOADER_DRIVER_OVERRIDE=zink
AMD_OVERRIDE_DEVICE_ID=0x731F
HSA_OVERRIDE_GFX_VERSION=10.1.0
RADV_PERFTEST=sam
MESA_SHADER_CACHE_MAX_SIZE=4G
```

> `RADV_DEBUG=nohiz` desativa HIZ (problemático na BC-250). `nocompute` previne crashes com compute shaders — no **Mesa 25.1+** já é desabilitado automaticamente, mas não faz mal manter. `AMD_OVERRIDE_DEVICE_ID=0x731F
HSA_OVERRIDE_GFX_VERSION=10.1.0
RADV_PERFTEST=sam
MESA_SHADER_CACHE_MAX_SIZE=4G` faz o Vulkan/DXVK identificar como Navi10.

### Unified heap (`/etc/drirc`)

```xml
<driconf>
    <device>
        <application name="Default">
            <option name="radv_enable_unified_heap_on_apu" value="true" />
        </application>
    </device>
</driconf>
```

### Kernel parameters

```
quiet mitigations=off nmi_watchdog=1 amdgpu.sg_display=0 \
amdgpu.ppfeaturemask=0xfffd7fff amdgpu.vm_fragment_size=9
```

> **Não usar `nowatchdog`** — o watchdog de hardware (SP5100 TCO) é essencial pra recuperar de hard locks causados pelo driver amdgpu. Zero impacto em performance.

> `amdgpu.sg_display=0` — só obrigatório em kernels < 6.10. Em kernels modernos é opcional mas inofensivo. `mitigations=off` dá +10-18 FPS (desativa mitigações Spectre/Meltdown — só usar em sistema dedicado a gaming).

---

## 5. Fan Control

### Driver nct6687

O driver padrão `nct6683` é read-only. Instalar o driver com suporte a escrita:

```bash
paru -S nct6687d-dkms-git
```

Blacklist o driver antigo e carregar o novo:

```bash
# /etc/modprobe.d/nct6683-blacklist.conf
blacklist nct6683

# /etc/modules-load.d/nct6687.conf
nct6687
```

### CoolerControl

```bash
sudo pacman -S coolercontrol
sudo systemctl enable --now coolercontrold
```

No `/etc/coolercontrol/config.toml`, o perfil de fan curve otimizado:

```toml
[[profiles]]
uid = "fa04993c-0336-4a42-968c-0f4768c7ffe6"
name = "auto"
p_type = "Graph"
speed_profile = [[30.0, 25], [45.0, 35], [55.0, 50], [65.0, 70], [72.0, 85], [80.0, 100], [90.0, 100]]
temp_source = { temp_name = "temp1", device_uid = "<UID-DO-K10TEMP>" }
temp_min = 20.0
temp_max = 100.0
function_uid = "0"
```

> **Nota**: O `device_uid` do k10temp é gerado por máquina. Use a UI do CoolerControl pra pegar o UID correto, ou cheque `/etc/coolercontrol/config.toml` após a primeira execução.

**Curva de fan:**
| Temp | Fan% |
|------|------|
| 30°C | 25%  |
| 45°C | 35%  |
| 55°C | 50%  |
| 65°C | 70%  |
| 72°C | 85%  |
| 80°C | 100% |

---

## 6. Streaming Headless (Sunshine + Moonlight)

### Dependências

```bash
sudo pacman -S sunshine labwc foot fuzzel seatd xdg-desktop-portal-wlr
sudo systemctl enable --now seatd
sudo usermod -aG input,seat $USER
```

### Compositor headless (`~/.config/systemd/user/labwc-headless.service`)

```ini
[Unit]
Description=Headless labwc Wayland compositor
After=graphical.target

[Service]
Type=simple
Environment=WLR_BACKENDS=headless,libinput
Environment=WLR_HEADLESS_OUTPUTS=1
Environment=XDG_SESSION_TYPE=wayland
ExecStart=/usr/bin/labwc
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

### labwc autostart (`~/.config/labwc/autostart`)

```bash
# Sunshine
(sleep 3 && sunshine) &
```

### labwc keybinds (`~/.config/labwc/rc.xml`)

```xml
<?xml version="1.0"?>
<labwc_config>
  <keyboard>
    <keybind key="C-A-e">
      <action name="Execute" command="foot"/>
    </keybind>
    <keybind key="C-A-r">
      <action name="Execute" command="fuzzel"/>
    </keybind>
    <keybind key="C-A-q">
      <action name="Close"/>
    </keybind>
    <keybind key="C-A-f">
      <action name="ToggleFullscreen"/>
    </keybind>
    <keybind key="A-Tab">
      <action name="NextWindow"/>
    </keybind>
  </keyboard>
  <theme>
    <name>Clearlooks-3.4</name>
  </theme>
</labwc_config>
```

### Sunshine config (`~/.config/sunshine/sunshine.conf`)

```ini
encoder = software
adapter_name = /dev/dri/renderD128
output_name = 0

sw_preset = ultrafast
sw_tune = zerolatency

### Sunshine Real-time Priority

Permite que o Sunshine tenha prioridade de CPU sobre o jogo, garantindo stream fluido mesmo em 100% load:

```bash
sudo setcap cap_sys_nice+ep /usr/bin/sunshine
```
```

> Sem VCN: A BC-250 não tem hardware de encoding (firmware bloqueado pela Sony). `encoder = software` (libx264) é a única opção. O preset `ultrafast` usa ~40% CPU durante stream. Para melhor qualidade com mais CPU, use `superfast` (~55%) ou `faster`.

### Habilitar serviços

Apenas o pipewire sobe com o sistema. Os serviços de display (labwc) **não sobem automaticamente** — são controlados pelos scripts abaixo.

```bash
systemctl --user enable pipewire pipewire-pulse wireplumber
```

### Comandos rápidos

Copiar os scripts do repo para `~/.local/bin/` e torná-los executáveis:

```bash
cp stream-on stream-off screen-on screen-off hud-on hud-off ~/.local/bin/
chmod +x ~/.local/bin/stream-on ~/.local/bin/stream-off \
         ~/.local/bin/screen-on ~/.local/bin/screen-off \
         ~/.local/bin/hud-on ~/.local/bin/hud-off
```

#### stream-on / stream-off (modo headless — Moonlight)

```bash
stream-on              # 1920x1080@60 (padrão)
stream-on 2560x1440@60
stream-on 1280x720@60
stream-off
```

#### screen-on / screen-off (monitor físico conectado)

```bash
screen-on              # 1920x1080@60 no primeiro output detectado
screen-on 2560x1440@60
screen-on 1920x1080@60 HDMI-A-1   # especificar output manualmente
screen-off

# Ver outputs disponíveis:
WAYLAND_DISPLAY=wayland-0 wlr-randr
```

> `screen-on` para o headless e sobe com DRM (monitor físico). `stream-on` faz o inverso.

### HUD (MangoHud on/off)

Criar `~/.local/bin/hud-on`:

```bash
#!/bin/sh
mkdir -p ~/.config/MangoHud
cat > ~/.config/MangoHud/MangoHud.conf << 'EOF'
gpu_temp
cpu_temp
gpu_power
ram
vram
fps
frametime=0
frame_timing=1
position=top-left
font_size=24
EOF
echo "HUD ON"
```

Criar `~/.local/bin/hud-off`:

```bash
#!/bin/sh
mkdir -p ~/.config/MangoHud
cat > ~/.config/MangoHud/MangoHud.conf << 'EOF'
no_display
EOF
echo "HUD OFF"
```

```bash
chmod +x ~/.local/bin/hud-on ~/.local/bin/hud-off
```

> Jogos no Steam precisam ter `mangohud %command%` nas Launch Options. O `hud-on`/`hud-off` liga e desliga o overlay. Precisa reiniciar o jogo pra aplicar.

### Conectar

No Moonlight, adicionar host por `ps6.local` ou pelo IP da máquina.

---

## 7. Proton-GE

```bash
mkdir -p ~/.local/share/Steam/compatibilitytools.d/
cd ~/.local/share/Steam/compatibilitytools.d/
curl -sLO "https://github.com/GloriousEggroll/proton-ge-custom/releases/latest/download/GE-ProtonX-XX.tar.gz"
tar -xzf GE-Proton*.tar.gz && rm GE-Proton*.tar.gz
```

Reiniciar Steam. Selecionar em cada jogo: Properties → Compatibility → GE-Proton.

---

## 8. Serviços desativados (otimização)

```bash
# Sistema (Mascarar para evitar subir no boot)
sudo systemctl mask lightdm accounts-daemon power-profiles-daemon upower
sudo systemctl mask plymouth-start.service plymouth-quit.service \
  plymouth-quit-wait.service plymouth-read-write.service \
  lvm2-monitor.service earlyoom.service

# User (Mascarar serviços de background desnecessários)
systemctl --user mask gvfs-daemon gvfs-afc-volume-monitor gvfs-gphoto2-volume-monitor \
  gvfs-mtp-volume-monitor gvfs-udisks2-volume-monitor gvfs-metadata \
  at-spi-dbus-bus xdg-document-portal
```

> **Nota**: `avahi-daemon` e `tailscaled` devem ser mantidos ativos para Moonlight/Sunshine e acesso remoto. `earlyoom` foi desativado para evitar que mate processos de IA pesados em picos de memória.

---

## 9. Avahi (mDNS)

Desabilitar IPv6 pra evitar problemas de resolução no Moonlight:

```bash
sudo sed -i 's/#use-ipv6=yes/use-ipv6=no/' /etc/avahi/avahi-daemon.conf
sudo systemctl restart avahi-daemon
```

---

## 10. Monitoramento

### Sensores disponíveis

| Sensor | Chip | O que monitora |
|---|---|---|
| `Tctl` | k10temp | Temperatura CPU (Zen 2) |
| `edge` | amdgpu | Temperatura GPU |
| `PPT` | amdgpu | Consumo GPU (watts) |
| `sclk` | amdgpu | Clock GPU (MHz) |
| `vddgfx` | amdgpu | Voltagem GPU core |
| `vddnb` | amdgpu | Voltagem northbridge |
| `Pump Fan` | nct6686 | RPM do ventilador (fan2) |
| `CPU` | nct6686 | Temperatura CPU (via SuperIO) |
| `System` | nct6686 | Temperatura da placa |
| `VRM MOS` | nct6686 | Temperatura VRM |
| `Composite` | nvme | Temperatura NVMe |

### Temperaturas seguras

| Condição | GPU | CPU | Consumo |
|---|---|---|---|
| Idle | 45-55°C | 45-55°C | 50-70W |
| Gaming leve | 60-75°C | 55-70°C | 100-150W |
| Gaming pesado | 70-85°C | 65-80°C | 150-200W |
| Stress test | 80-86°C | 75-85°C | 200-235W |
| **Throttle** | — | **95°C** | — |
| **Shutdown** | — | **105°C** | — |

### Comandos rápidos de monitoramento

```bash
# Todos os sensores
sensors

# Monitorar em tempo real
watch -n 1 sensors

# Temperatura GPU
cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input | awk '{print $1/1000 "°C"}'

# Consumo GPU
cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_average | awk '{print $1/1000000 "W"}'

# Clock GPU
cat /sys/class/drm/card0/device/pp_dpm_sclk

# Monitor GPU (terminal)
nvtop

# Monitor GPU AMD
radeontop
```

### MangoHud (overlay em jogos)

Config em `~/.config/MangoHud/MangoHud.conf`:

```ini
gpu_temp
cpu_temp
gpu_power
ram
vram
fps
frametime=0
frame_timing=1
position=top-left
font_size=24
```

Usar no Steam: Launch Options → `mangohud %command%`

### Script de log de temperatura

```bash
#!/bin/bash
# Salvar como ~/monitor-temps.sh && chmod +x ~/monitor-temps.sh
LOGFILE="$HOME/bc250-temps.log"
echo "Timestamp,GPU_Temp,CPU_Temp,GPU_Power" > "$LOGFILE"
while true; do
    GPU_TEMP=$(cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input 2>/dev/null | awk '{print $1/1000}')
    CPU_TEMP=$(sensors k10temp-pci-00c3 -u 2>/dev/null | grep temp1_input | awk '{print $2}')
    GPU_POWER=$(cat /sys/class/drm/card0/device/hwmon/hwmon*/power1_average 2>/dev/null | awk '{print $1/1000000}')
    echo "$(date +%s),$GPU_TEMP,$CPU_TEMP,$GPU_POWER" >> "$LOGFILE"
    sleep 5
done
```

---

## 11. Overclock da GPU

A GPU vai de 1000 MHz (idle) até 2200 MHz (load) com os safe-points do governor.

### Limites conhecidos

| Frequência | Voltagem | Status |
|---|---|---|
| 1000 MHz | 800 mV | Mínimo (idle) |
| 2000 MHz | 960 mV | Safe pra todas as placas |
| 2175 MHz | 1025 mV | Safe pra maioria |
| 2200 MHz | 1030 mV | Configurado atual |
| 2230 MHz | — | Máximo (requer patch de kernel) |

### Teste manual de OC

```bash
# Parar governor
sudo systemctl stop cyan-skillfish-governor-smu

# Testar frequência
echo "vc 0 2100 1050" > /sys/devices/pci0000:00/0000:00:08.1/0000:01:00.0/pp_od_clk_voltage

# Rodar benchmark por 30+ minutos
# Se estável, atualizar config.toml com os novos valores

# Reiniciar governor
sudo systemctl start cyan-skillfish-governor-smu
```

### Se crashar durante gaming

- Aumentar voltagem (subir max pra 1050 mV)
- Reduzir frequência máxima (tentar 1900 MHz)
- Verificar temperaturas (deve estar <85°C)

---

## 12. Overclock da CPU (SMU)

A CPU stock roda a 3500 MHz. Com o tool `bc250-smu-oc` é possível subir via SMU.

### Instalar

```bash
sudo pacman -S pipx
pipx ensurepath
git clone https://github.com/bc250-collective/bc250_smu_oc
cd bc250_smu_oc
pipx install .
```

### Testar (não persiste)

```bash
# Testar 3800 MHz @ 1150 mV
bc250-detect --frequency 3800 --vid 1150

# Ajustar em steps de 100 MHz e 25 mV pra achar o sweet spot
```

### Aplicar (persiste até reboot)

```bash
# Adicionar -k pra manter após o teste
bc250-detect --frequency 3800 --vid 1150 -k
```

### Aplicar no boot

```bash
# Após encontrar valores estáveis, salvar config e habilitar serviço
bc250-apply --install overclock.conf
sudo systemctl enable --now bc250-smu-oc
```

### Configuração atual (`/etc/bc250-smu-oc.conf`)

```ini
[overclock]
frequency = 3800
scale = -29
max_temperature = 95
```

### Voltar ao stock (3500 MHz)

```bash
sudo systemctl stop bc250-smu-oc
sudo systemctl disable bc250-smu-oc
# Reboot pra resetar
```

### Regras de segurança

- **Nunca exceder 1300 mV** — pode danificar a CPU
- **Cada chip é diferente** — valores ótimos variam entre placas
- **Manter temperatura abaixo de 85°C** em jogos antes de fazer OC
- Testar com stress por pelo menos 5 minutos antes de aplicar permanente
- Ajustar em steps de 100 MHz (freq) e 25 mV (voltagem)

---

## 13. Resiliência e Auto-Recovery

Por padrão o CachyOS trava para sempre em kernel panic. Essas configs garantem que o sistema sempre reinicia sozinho.

### Kernel panic → reboot automático

```bash
# /etc/sysctl.d/99-resilience.conf
sudo cp config/99-resilience.conf /etc/sysctl.d/99-resilience.conf
sudo sysctl -p /etc/sysctl.d/99-resilience.conf
```

O arquivo configura:
- `kernel.panic = 10` — reboot 10s após kernel panic (driver AMD crashou, etc.)
- `kernel.panic_on_oops = 1` — trata kernel oops como panic (evita sistema "meio vivo")
- `kernel.hung_task_panic = 1` — reboot se processo do kernel travar por 120s

### Watchdog de hardware (SP5100 TCO)

O watchdog de hardware é um timer no chipset AMD que força reboot em hard locks (quando nem o kernel responde). **Não adicionar `nowatchdog` nos kernel parameters.**

> ⚠️ O `sp5100_tco` vem **blacklisted por padrão** no CachyOS (`/usr/lib/modprobe.d/blacklist.conf`). O `blacklist` do kmod é aditivo e não pode ser overridden por outro arquivo em `/etc/modprobe.d/`. A solução é um serviço systemd que carrega o módulo no boot:

```bash
# /etc/systemd/system/sp5100-watchdog.service
[Unit]
Description=Load SP5100 TCO watchdog module
DefaultDependencies=no
After=systemd-modules-load.service
Before=sysinit.target

[Service]
Type=oneshot
ExecStart=/sbin/modprobe sp5100_tco
RemainAfterExit=yes

[Install]
WantedBy=sysinit.target
```

```bash
sudo systemctl enable sp5100-watchdog.service
```

Configurar o systemd pra alimentar o watchdog:
```bash
# /etc/systemd/system.conf — adicionar:
WatchdogDevice=/dev/watchdog0
```

Verificar se está ativo após reboot:
```bash
dmesg | grep -i watchdog
# Deve mostrar: sp5100_tco ou similar

cat /sys/class/watchdog/watchdog0/identity
# Deve mostrar: SP5100 TCO timer

cat /sys/class/watchdog/watchdog0/status
# 0x8100 = ativo e sendo alimentado pelo systemd
```

### NMI Watchdog (Non-Maskable Interrupt)

O NMI watchdog detecta hard locks que nem o SP5100 consegue pegar — quando o CPU trava em um loop que o watchdog de hardware não percebe. Usa um contador PMU (Performance Monitoring Unit) do CPU pra detectar se o sistema parou de responder.

O CachyOS desabilita por padrão (`nmi_watchdog=0`) por performance. Custo de ativar: ~1% CPU.

Adicionar ao kernel cmdline (`/boot/refind_linux.conf`):
```
nmi_watchdog=1
```

Verificar:
```bash
cat /proc/sys/kernel/nmi_watchdog
# Deve mostrar: 1
```

### systemd watchdog

Reinicia o sistema se o próprio systemd travar:

```bash
# /etc/systemd/system.conf — adicionar/descomentar:
RuntimeWatchdogSec=30
RebootWatchdogSec=10min
```

```bash
sudo systemctl daemon-reexec
```

### earlyoom (OOM killer antes do colapso)

O OOM killer padrão age tarde demais. `earlyoom` age quando ainda há memória suficiente pro kernel funcionar, matando o processo mais pesado (ex: ComfyUI) antes de travar tudo:

```bash
sudo pacman -S earlyoom
sudo systemctl enable --now earlyoom
```

Config padrão já funciona bem. Vai matar o processo que estiver comendo mais memória quando o sistema chegar a ~5% de RAM livre.

---

## 14. Compatibilidade de Jogos e Performance

### Performance esperada

Equivalente a uma **RX 6600** para rasterização. ~70-80% da performance de um PS5.

| Resolução | Settings | FPS esperado |
|-----------|----------|-------------|
| 1080p High | Sem RT | 60–100+ |
| 1080p Medium | Sem RT | 80–120+ |
| 1080p + RT | Lighting only | 30–60 |
| 1440p + FSR | Qualidade | Jogável na maioria |

### Jogos testados

| Jogo | FPS | Config | Notas |
|------|-----|--------|-------|
| Cyberpunk 2077 | 70–90 | 1080p High + FSR | +18 FPS com `mitigations=off` |
| Cyberpunk 2077 | 50–60 | 1080p High + FSR + RT | RT lighting only |
| Cyberpunk 2077 | 100+ | 1080p Ultra + FSR 3.1 | |
| The Last of Us Part I | 60 | 1080p Medium-High | 90–100°C na compilação de shaders |
| Control | 40 | 1080p + RT full | |
| Devil May Cry 5 | 100+ | 1080p High | Excelente otimização |
| Detroit: Become Human | 60 | 1080p Medium | Travado em 60 |
| Red Dead Redemption 2 | 45+ | Benchmark | Usar `-useMaximumSettings` |
| CS2 | 100+ | 1080p | Alta latência GDDR6 pode afetar |
| Portal RTX | 40 | 720p | RT real (Mesa 25.2+) |
| Tears of the Kingdom (Ryujinx) | 20 | — | Limitação da placa nesse jogo |

### Jogos incompatíveis (limitação de hardware)

- **FF7 Rebirth** — requer mesh shaders (não suportado no GFX1013)
- **Doom: The Dark Ages** — requer Vulkan Fragment Shading Rate
- **Fortnite** — Easy Anti-Cheat não suporta Linux

### Steam Launch Options recomendadas

```bash
# Básico
RADV_DEBUG=nohiz %command%

# Com overlay
RADV_DEBUG=nohiz mangohud gamemoderun %command%
```

### Company of Heroes 3

Requer VRAM split de pelo menos 4 GB (512 MB causa artefatos/crashes).

---

## 15. LLM Inference (llama.cpp via Vulkan)

A BC-250 roda modelos de linguagem via backend Vulkan do llama.cpp — sem ROCm (experimental e com problemas).

```bash
# Baixar binário pré-compilado com Vulkan
wget https://github.com/ggerganov/llama.cpp/releases/latest/download/llama-*-bin-ubuntu-vulkan-x64.zip
unzip llama-*.zip && cd build/bin

# Variável necessária para evitar OOM em alocações grandes
export GGML_VK_FORCE_MAX_ALLOCATION_SIZE=2000000000  # 2GB por chunk

# Rodar servidor
./llama-server --model /caminho/modelo.gguf --gpu-layers 99
```

**Performance esperada:**
- Modelo 8B quantizado (Q4): ~60 tokens/seg
- VRAM visível pelo Vulkan: ~10 GB (dos 16 GB totais — limitação conhecida)
- Modelos 70B+: provável OOM mesmo quantizados

**ROCm:** Suporte experimental e incompleto para GFX1013 — `rocBLAS` não tem kernels pré-compilados para essa arquitetura. Usar Vulkan.

---

## Checklist de verificação

```bash
# Kernel
uname -r

# Mesa
pacman -Q mesa vulkan-radeon

# GPU reconhecida
vulkaninfo 2>&1 | grep deviceName
# Expected: AMD BC-250 (RADV GFX1013)

# VRAM total
vulkaninfo 2>&1 | grep -A2 "memoryHeaps\[0\]"
# Expected: ~14.5 GB

# Governor
systemctl status cyan-skillfish-governor-smu
cat /sys/class/drm/card0/device/pp_dpm_sclk

# Fan control
systemctl status coolercontrold
sensors | grep "Pump Fan"

# Sunshine
systemctl --user status labwc-headless
ps aux | grep sunshine

# Audio
systemctl --user status pipewire

# Seat
systemctl status seatd

# Resiliência
sysctl kernel.panic          # deve ser 10
sysctl kernel.panic_on_oops  # deve ser 1
systemctl status earlyoom
```

---

## Uso de recursos (idle, sem jogo)

| Componente | CPU | RAM |
|---|---|---|
| labwc | ~4% | ~130 MB |
| Sunshine (idle) | ~2% | ~120 MB |
| Sunshine (stream) | ~40% | ~200 MB |
| PipeWire | ~1% | ~40 MB |
| **Total (sem Steam)** | ~7% | ~400 MB |
| Steam (aberto) | ~10% | ~2 GB |

---

## Referências
## Referências

- [AMD BC-250 Docs](https://elektricm.github.io/amd-bc250-docs/)
- [Sunshine](https://app.lizardbyte.dev/)
- [Moonlight](https://moonlight-stream.org/)
- [CoolerControl](https://gitlab.com/coolercontrol/coolercontrol)
- [GE-Proton](https://github.com/GloriousEggroll/proton-ge-custom)
