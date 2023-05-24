import datetime
import socket
import threading
import time
from functools import reduce
from dateutil import parser

client_data = {}

def receive_clock_time(connector, address):
    while True:
        clock_time_string = connector.recv(1024).decode()
        clock_time = parser.parse(clock_time_string)
        clock_time_diff = datetime.datetime.now() - clock_time
        client_data[address] = {
            "clock_time": clock_time,
            "time_difference": clock_time_diff,
            "connector": connector
        }
        print(f"Client Data updated with: {address}\n")
        time.sleep(5)

def connect_clients(master_server):
    while True:
        master_slave_connector, addr = master_server.accept()
        slave_address = f"{addr[0]}:{addr[1]}"
        print(f"{slave_address} got connected successfully")
        current_thread = threading.Thread(target=receive_clock_time, args=(master_slave_connector, slave_address))
        current_thread.start()

def synchronize_clocks():
    while True:
        print("New synchronization cycle started.")
        print(f"Number of clients to be synchronized: {len(client_data)}")
        if len(client_data) > 0:
            average_clock_difference = calculate_average_clock_diff()
            for client_addr, client in client_data.items():
                try:
                    synchronized_time = datetime.datetime.now() + average_clock_difference
                    client['connector'].send(str(synchronized_time).encode())
                except Exception as e:
                    print(f"Something went wrong while sending synchronized time through {client_addr}")
        else:
            print("No client data. Synchronization not applicable.")
        print("\n")
        time.sleep(5)

def calculate_average_clock_diff():
    current_client_data = client_data.copy()
    time_difference_list = [client['time_difference'] for client in current_client_data.values()]
    sum_of_clock_difference = reduce(lambda x, y: x + y, time_difference_list, datetime.timedelta(0))
    average_clock_difference = sum_of_clock_difference / len(client_data)
    return average_clock_difference

def initiate_clock_server(port=8080):
    global master_server
    master_server = socket.socket()
    master_server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    print("Socket at master node created successfully\n")
    master_server.bind(('', port))
    master_server.listen(10)
    print("Clock server started...\n")
    print("Starting to make connections...\n")
    master_thread = threading.Thread(target=connect_clients, args=(master_server,))
    master_thread.start()
    print("Starting synchronization parallelly...\n")
    sync_thread = threading.Thread(target=synchronize_clocks)
    sync_thread.start()

if _name_ == '_main_':
    initiate_clock_server(port=8080)
```