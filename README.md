import socket
import struct
import requests
import time
import json
from flask import Flask, Response, stream_with_context
from flask_cors import CORS
from threading import Thread
import threading
import datetime 
# Define constants
SRS_SERVER_PORT = 29172
SRS_MAX_POINT = 2000
SRS_MAX_TARGET = 250

# Ip address
sourceIp = "192.168.30.1"

SRS_TARGET_STATUS_STANDING = 0
SRS_TARGET_STATUS_LYING = 1
SRS_TARGET_STATUS_SITTING = 2
SRS_TARGET_STATUS_FALL = 3
SRS_TARGET_STATUS_UNKNOWN = 4

# Target information structure
class SRS_TARGET_INFO:
    def __init__(self, posX=0.0, posY=0.0, status=SRS_TARGET_STATUS_UNKNOWN, id=0, reserved=[0.0, 0.0, 0.0]):
        self.posX = posX
        self.posY = posY
        self.status = status
        self.id = id
        self.reserved = reserved

magic_word_bytes = struct.pack('HHHH', 0x0201, 0x0403, 0x0605, 0x0807)

# Flask app for SSE
app = Flask(__name__)
CORS(app, resources={r"/stream": {"origins": "http://localhost:3000"}})
# Shared data
target_info = None

def memset(buffer, value, size):
    for i in range(size):
        buffer[i] = value

# SSE endpoint
@app.route('/stream')
def stream():
    def event_stream():
        while True:
            if target_info:
                data = {
                    "x": target_info.posX,
                    "y": target_info.posY,
                    "state": target_info.status,
                    "time": time.strftime("%Y-%m-%d %H:%M:%S")
                }
                print(data)
                yield f"data: {json.dumps(data)}\n\n"
            threading.Timer(5, stream).start()
           # time.sleep(1)  # Send data every 5 minutes

    return Response(stream_with_context(event_stream()), content_type='text/event-stream')

def read_packet(sock, packet_buffer, size_of_buffer):
    read_num = 0
    magic_number = 0xABCD4321
    packet_buffer[:] = bytes([0] * size_of_buffer)

    header = sock.recv(36)
    if len(header) != 36:
        return -1

    packet_number_bytes = header[4:8]
    packet_number = struct.unpack('I', packet_number_bytes)[0]

    if packet_number != magic_number:
        return 0

    packet_size_bytes = header[16:20]
    packet_size = struct.unpack('I', packet_size_bytes)[0]

    packet_read = 0

    while packet_read < packet_size:
        received_data = sock.recv(packet_size - packet_read)
        if len(received_data) <= 0:
            return -1
        packet_buffer[packet_read:packet_read+len(received_data)] = received_data
        packet_read += len(received_data)

    return packet_size

def sendToUnity(id, posX, posY, status):
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    serverAddressPort = ("127.0.0.1", 5051)
    message = "{}|{}|{}|{}|{}".format(id, posX, posY, status, 0)
    sock.sendto(message.encode(), serverAddressPort)
    #print(message)

def send_status_to_server(status):
    url = "http://localhost:8080/status"
    data = {"state": status}
    try:
        response = requests.post(url, json=data)
        if response.status_code == 200:
            print("Status successfully sent to server.")
        else:
            print("Failed to send status to server. Status code:", response.status_code)
    except Exception as e:
        print("An error occurred while sending status to server:", e)
def send_log_to_server(status,posX, posY):
    url = "http://localhost:8080/log/log"
    now = datetime.datetime.now()
    c_datetime = now.strftime("%H:%M:%S")
    if(status!="FALL"):
        type = "normal"
    else : type = "emergency"
    if(target_info.posX>-0.1 and target_info.posY<1):
        area = 1
    elif(target_info.posX>-1 and target_info.posY<2):
        area = 2
    else:
        area = 3
    data = {
        "patientid":"dusco01",
        "content": status,
        "type" : type,
        "datetime" : c_datetime,
        "x": target_info.posX,
        "y": target_info.posX,
        "area": area,
    }
    try:
        response = requests.post(url, json=data)
        if response.status_code == 200:
            print(data)
        else:
            print("Failed to send status to server. Status code:", response.status_code)
    except Exception as e:
        print("An error occurred while sending status to server:", e)
def main():
    global target_info
    packet_buffer = bytearray(512000)
    buffer_idx = 0
    packet_size = 0
    magic_word = struct.pack('HHHH', 0x0201, 0x0403, 0x0605, 0x0807)
    point_num = 0
    target_id_per_point = bytearray(SRS_MAX_POINT)

    has_target = 0
    target_num = 0
    target = [None] * SRS_MAX_TARGET

    sock = None
    count = 0
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        print("Connecting to {}...".format(sourceIp))
        serveraddr = (sourceIp, SRS_SERVER_PORT)
        sock.connect(serveraddr)
        print("Connected.")
    except Exception as e:
        print("Connection failed:", e)

    while True:
        has_target = 0
        packet_buffer = bytearray(512000)
        packet_size = read_packet(sock, packet_buffer, len(packet_buffer))
        if packet_size < 0:
            print("Connection closed.")
            break
        elif packet_size == 0:
            continue

        buffer_idx = 0

        if packet_buffer[buffer_idx:buffer_idx + len(magic_word)] != magic_word_bytes:
            print("Invalid magic word.")
            break

        buffer_idx += len(magic_word)
        frame_count = struct.unpack_from('I', packet_buffer, buffer_idx)[0]
        buffer_idx += 4

        point_num = struct.unpack_from('I', packet_buffer, buffer_idx)[0]
        buffer_idx += 4

        for step in range(point_num):
            buffer_idx += 20

        if packet_buffer[buffer_idx:buffer_idx + len(magic_word)] == magic_word:
            has_target = 1
        elif packet_size > 48020 and packet_buffer[48020:48020 + len(magic_word)] == magic_word:
            has_target = 2

        if has_target:
            for step in range(point_num):
                target_id_per_point[step] = packet_buffer[buffer_idx]
                buffer_idx += 1

        if has_target == 2:
            buffer_idx = 48020

        if has_target:
            buffer_idx += len(magic_word)
            frame_count = struct.unpack_from('I', packet_buffer, buffer_idx)[0]
            buffer_idx += 4

            target_num = struct.unpack_from('I', packet_buffer, buffer_idx)[0]
            buffer_idx += 4

            for step in range(target_num):
                try:
                    pos_x, pos_y, status, target_id, reserved_1, reserved_2, reserved_3 = struct.unpack_from('ffIIfff', packet_buffer, buffer_idx)
                    if step < len(target):
                        target_info = SRS_TARGET_INFO(pos_x, pos_y, status, target_id, [reserved_1, reserved_2, reserved_3])
                        target[step] = target_info
                    else:
                        print("skip target.")
                    buffer_idx += struct.calcsize('ffIIfff')
                except IndexError:
                    print("skip target")
                    continue
                        
            for step in range(target_num):
                status_str = "UNKNOWN"
                status = target[step].status
                if status == SRS_TARGET_STATUS_STANDING:
                    status_str = "STANDING"
                elif status == SRS_TARGET_STATUS_LYING:
                    status_str = "LYING"
                elif status == SRS_TARGET_STATUS_SITTING:
                    status_str = "SITTING"
                elif status == SRS_TARGET_STATUS_FALL:
                    status_str = "FALL"
                    send_status_to_server("fall")
                #sendToUnity(target[step].id, target[step].posX, target[step].posY, status_str)
                count += 1 
                if count % 1000 == 0:
                    send_log_to_server(status_str, target[step].posX, target[step].posY)
                

    sock.close()
    return 0

if __name__ == "__main__":
    # Start the SSE server in a separate thread
    server_thread = Thread(target=lambda: app.run(port=8081, debug=False, use_reloader=False))
    server_thread.start()

    # Run the main processing function
    main()
