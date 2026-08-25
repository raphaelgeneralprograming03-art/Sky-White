import numpy as np
import matplotlib.pyplot as plt
import matplotlib.animation as animation

# ==========================================
# CONFIGURACAO DA SIMULACAO DE DRONES (IA vs COMERCIAL)
# ==========================================

# Parametros Temporais
frames = 200
t = np.linspace(0, 10, frames)

# Curva de Ruido Eletromagnetico (0 dB a 100 dB)
db_noise = np.linspace(0, 100, frames)

# Eficiencia (%)
# Drone Comercial colapsa exponencialmente com o rui­do
comm_eff = 100 / (1 + np.exp((db_noise - 40) / 8))
# Seu Sistema se estabiliza em ~94% (processamento 100% local)
sys_eff = np.full(frames, 94.0) + np.random.normal(0, 0.2, frames)

# Trajetoria do Alvo (Visao Computacional)
target_x = 8 + 0.6 * np.sin(2 * t)
target_y = 10 - 0.5 * t

# Trajetoria Drone Comercial
comm_x = np.zeros(frames)
comm_y = np.zeros(frames)
for i in range(frames):
    if db_noise[i] < 35:
        comm_x[i] = 2 + 0.2 * np.sin(2 * t[i])
        comm_y[i] = 10 - 0.5 * t[i]
    else:
        # Perda de link / Deriva e Queda Livre
        drift = (db_noise[i] - 35) * 0.05
        comm_x[i] = comm_x[i-1] + np.random.normal(0.05, 0.1)
        comm_y[i] = comm_y[i-1] - 0.15 - drift * 0.02

# Trajetoria Seu Drone (Controle Preditivo E(t) = P_alvo - P_centro)
sys_x = np.zeros(frames)
sys_y = np.zeros(frames)
e_t_x = np.zeros(frames)
e_t_y = np.zeros(frames)

for i in range(frames):
    # Erro de pixels/posicao reduzido via torque preditivo
    e_x = target_x[i] - (sys_x[i-1] if i > 0 else 8)
    e_y = target_y[i] - (sys_y[i-1] if i > 0 else 10)
    e_t_x[i] = e_x
    e_t_y[i] = e_y
    
    # Atualizacao de posicao via Torque Mecanico local
    sys_x[i] = (sys_x[i-1] if i > 0 else 8) + 0.85 * e_x + np.random.normal(0, 0.02)
    sys_y[i] = (sys_y[i-1] if i > 0 else 10) + 0.85 * e_y + np.random.normal(0, 0.02)

# ==========================================
# CRIACAO DA INTERFACE GRAFICA (MATPLOTLIB)
# ==========================================

plt.style.use('dark_background')
fig = plt.figure(figsize=(14, 8), dpi=100)
fig.suptitle('Simulacao Dinamica: Drone Comercial vs. IA Preditiva Local [E(t) & Rui­do dB]', fontsize=14, fontweight='bold', color='#00E5FF')

# Subplot 1: Espaço de Voo e Trajetoria 2D
ax_map = plt.subplot2grid((2, 2), (0, 0), rowspan=2)
ax_map.set_title('Arena de Voo & Rastreamento em Tempo Real', color='white', fontsize=11)
ax_map.set_xlim(-1, 12)
ax_map.set_ylim(-2, 12)
ax_map.set_xlabel('Posicao X (m)')
ax_map.set_ylabel('Posicao Y (m) [Altitude]')
ax_map.grid(True, linestyle='--', alpha=0.3)

# Subplot 2: Grafico de Ruido dB vs Eficiencia (%)
ax_eff = plt.subplot2grid((2, 2), (0, 1))
ax_eff.set_title('Resistencia a Ruido Eletromagnetico (dB)', color='white', fontsize=11)
ax_eff.set_xlim(0, 100)
ax_eff.set_ylim(0, 110)
ax_eff.set_xlabel('Interferencia Externa (dB)')
ax_eff.set_ylabel('Eficiencia Operacional (%)')
ax_eff.grid(True, linestyle='--', alpha=0.3)

# Subplot 3: Vetor de Erro Dinamico E(t)
ax_err = plt.subplot2grid((2, 2), (1, 1))
ax_err.set_title('Vetor de Erro Dinamico E(t) = P_alvo - P_centro', color='white', fontsize=11)
ax_err.set_xlim(0, 10)
ax_err.set_ylim(-1.5, 1.5)
ax_err.set_xlabel('Tempo (s)')
ax_err.set_ylabel('Erro Pixel/Distancia (m)')
ax_err.grid(True, linestyle='--', alpha=0.3)

# Elementos Visuais na Arena
line_target, = ax_map.plot([], [], 'r--', alpha=0.6, label='Alvo Visual (P_alvo)')
point_target, = ax_map.plot([], [], 'ro', markersize=8)

line_comm, = ax_map.plot([], [], '#FF3366', alpha=0.7, label='Drone Comercial (Dependente de GPS/Radio)')
point_comm, = ax_map.plot([], [], 'o', color='#FF3366', markersize=10)

line_sys, = ax_map.plot([], [], '#00FFCC', alpha=0.9, linewidth=2, label='Seu Drone (IA Local + Controle Preditivo)')
point_sys, = ax_map.plot([], [], 's', color='#00FFCC', markersize=10)

# Linhas de Grafico de Eficiencia
line_eff_comm, = ax_eff.plot([], [], color='#FF3366', linewidth=2, label='Drone Comercial')
line_eff_sys, = ax_eff.plot([], [], color='#00FFCC', linewidth=2.5, label='Seu Sistema (Estavel 94%)')

# Linhas de Grafico de Erro E(t)
line_err_x, = ax_err.plot([], [], color='#00FFCC', label='Erro Ex(t)')
line_err_y, = ax_err.plot([], [], color='#FFCC00', label='Erro Ey(t)')

# Anotacoes e Caixas de Texto
info_box = ax_map.text(0.02, 0.93, '', transform=ax_map.transAxes, bbox=dict(boxstyle='round', facecolor='#111122', alpha=0.8))

ax_map.legend(loc='lower left', fontsize=8)
ax_eff.legend(loc='upper right', fontsize=8)
ax_err.legend(loc='upper right', fontsize=8)

def init():
    line_target.set_data([], [])
    point_target.set_data([], [])
    line_comm.set_data([], [])
    point_comm.set_data([], [])
    line_sys.set_data([], [])
    point_sys.set_data([], [])
    line_eff_comm.set_data([], [])
    line_eff_sys.set_data([], [])
    line_err_x.set_data([], [])
    line_err_y.set_data([], [])
    info_box.set_text('')
    return line_target, point_target, line_comm, point_comm, line_sys, point_sys, line_eff_comm, line_eff_sys, line_err_x, line_err_y, info_box

def update(frame):
    # Atualiza Arena de Voo
    line_target.set_data(target_x[:frame], target_y[:frame])
    point_target.set_data([target_x[frame]], [target_y[frame]])
    
    line_comm.set_data(comm_x[:frame], comm_y[:frame])
    point_comm.set_data([comm_x[frame]], [comm_y[frame]])
    
    line_sys.set_data(sys_x[:frame], sys_y[:frame])
    point_sys.set_data([sys_x[frame]], [sys_y[frame]])
    
    # Atualiza Eficiencia
    line_eff_comm.set_data(db_noise[:frame], comm_eff[:frame])
    line_eff_sys.set_data(db_noise[:frame], sys_eff[:frame])
    
    # Atualiza Erro E(t)
    line_err_x.set_data(t[:frame], e_t_x[:frame])
    line_err_y.set_data(t[:frame], e_t_y[:frame])
    
    # Texto Informativo Dinamico
    status_comm = "CRAOTICO / FALHA" if db_noise[frame] > 40 else "NORMAL"
    status_text = (
        f"Ruido Eletromagnetico: {db_noise[frame]:.1f} dB\n"
        f"Eficiencia Comercial: {comm_eff[frame]:.1f}% [{status_comm}]\n"
        f"Eficiencia Seu Sistema: {sys_eff[frame]:.1f}% [OPERACIONAL - 100% LOCAL]\n"
        f"Erro E(t) Vector: ({e_t_x[frame]:.2f}, {e_t_y[frame]:.2f})"
    )
    info_box.set_text(status_text)
    
    return line_target, point_target, line_comm, point_comm, line_sys, point_sys, line_eff_comm, line_eff_sys, line_err_x, line_err_y, info_box

ani = animation.Funcanimation(fig, update, frames=frames, init_func=init, interval=50, blit=True)
plt.tight_layout()
plt.show()
