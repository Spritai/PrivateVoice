# PrivateVoice





import socket
import threading

TCP_PORT = 5000
UDP_PORT = 5001
HOST = '0.0.0.0'

clients = {}  # {socket: pseudo}
udp_clients = set()

def broadcast_user_list():
    """Envoie la liste des pseudos à tous les clients"""
    user_names = ",".join(clients.values())
    list_msg = f"LIST:{user_names}".encode('utf-8')
    for sock in list(clients.keys()):
        try:
            sock.send(list_msg)
        except:
            pass

def handle_tcp(client_socket, addr):
    print(f"[TCP] Connexion de {addr}")
    while True:
        try:
            data = client_socket.recv(1024)
            if not data:
                break
            
            msg = data.decode('utf-8')

            # 1. Gérer le PING (Silencieux)
            if msg == "PING":
                client_socket.send("PONG".encode('utf-8'))
                continue # On s'arrête ici, on ne diffuse pas

            # 2. Gérer le PSEUDO (Silencieux)
            if msg.startswith("NAME:"):
                pseudo = msg.split(":")[1]
                clients[client_socket] = pseudo
                broadcast_user_list()
                continue # On ne diffuse pas le message technique

            # 3. Diffuser les messages de CHAT classiques
            # On ne diffuse que si le client a déjà un pseudo enregistré
            if client_socket in clients:
                for sock in list(clients.keys()):
                    try:
                        sock.send(data)
                    except:
                        if sock in clients: del clients[sock]
        except:
            break
            
    if client_socket in clients:
        del clients[client_socket]
    client_socket.close()
    broadcast_user_list()
    print(f"[TCP] Déconnexion de {addr}")

def tcp_server():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind((HOST, TCP_PORT))
    s.listen(10)
    while True:
        client, addr = s.accept()
        threading.Thread(target=handle_tcp, args=(client, addr), daemon=True).start()

def udp_server():
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.bind((HOST, UDP_PORT))
    while True:
        try:
            data, addr = s.recvfrom(4096)
            if addr not in udp_clients:
                udp_clients.add(addr)
            for client_addr in list(udp_clients):
                if client_addr != addr:
                    try:
                        s.sendto(data, client_addr)
                    except:
                        udp_clients.discard(client_addr)
        except:
            pass

if __name__ == "__main__":
    print(f"Serveur lancé sur {HOST}...")
    threading.Thread(target=tcp_server, daemon=True).start()
    udp_server()
