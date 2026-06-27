
import socket
import threading
import json
import re

PROXY_HOST = "127.0.0.1"
PROXY_PORT = 8080
LISTEN_PORT = 8081

FAKE_CONFIG = {
    "code": 0,
    "data": {
        "version": "3.0.0",
        "config_ok": True,
        "force_update": False,
        "ios_support": True,
        "min_ios_version": "18.0"
    }
}

def handle_client(client_socket):
    try:
        request = client_socket.recv(8192)
        if not request:
            client_socket.close()
            return

        # KIỂM TRA REQUEST CHỨA CONFIG HOẶC VERSION
        if b"config" in request.lower() or b"version" in request.lower():
            response = f"HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n{json.dumps(FAKE_CONFIG)}"
            client_socket.send(response.encode())
        else:
            # CHUYỂN TIẾP QUA PROXYPIN
            proxy_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            proxy_socket.connect((PROXY_HOST, PROXY_PORT))
            proxy_socket.send(request)
            response = proxy_socket.recv(8192)

            # SỬA RESPONSE CHỨA VERSION (NẾU CÓ)
            try:
                body_start = response.find(b"\r\n\r\n") + 4
                if body_start > 4:
                    body = response[body_start:].decode('utf-8', errors='ignore')
                    if 'version' in body:
                        # SỬA LỖI REGEX: THAY THẾ TOÀN BỘ TRƯỜNG VERSION
                        new_body = re.sub(r'"version"\s*:\s*"[^"]*"', '"version":"3.0.0"', body)
                        response = response[:body_start] + new_body.encode()
            except Exception as e:
                print(f"[!] Lỗi sửa response: {e}")

            client_socket.send(response)
            proxy_socket.close()

    except Exception as e:
        print(f"[ERR] {e}")
    finally:
        client_socket.close()

def main():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(('0.0.0.0', LISTEN_PORT))
    server.listen(10)
    print(f"[*] Interceptor đang chạy trên cổng {LISTEN_PORT}")
    print("[*] Cấu hình proxy iPhone: IP máy tính, cổng 8081")
    while True:
        client, addr = server.accept()
        print(f"[+] Kết nối từ {addr}")
        threading.Thread(target=handle_client, args=(client,)).start()

if __name__ == "__main__":
    main()
