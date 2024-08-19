# Header
import json
import requests

# Specify IP address and port
ipAddress = "192.168.100.55"
url = "http://" + ipAddress + ":8080"

# Request item list
response = requests.get(url + "/item/list/")
if response.status_code == 200:
    print(json.dumps(response.json(), indent=4))
else:
    print("Error: ", response.status_code)

# Create list of Module and Channels
module_ids = []
channel_ids = []

response_data = response.json()
for item in response_data:
    if item["ItemType"] == "Module" and item["ItemName"] == "ICS425":
        module_ids.append(item["ItemId"])
    elif item["ItemType"] == "Channel" and item["ItemName"] == "ICS425":
        channel_ids.append(item["ItemId"])

print("Module IDs:", module_ids)
print("Channel IDs:", channel_ids)

# Get operation mode of first Module
response = requests.get(url + "/item/operationMode/", params={"itemId": module_ids[0]})
if response.status_code == 200:
    print(json.dumps(response.json(), indent=4))
else:
    print("Error: ", response.status_code)
    exit()

# Configure the sampling rate
for module_id in module_ids:
    response = requests.get(url + "/item/settings/", params={"itemId": module_id})
    if response.status_code == 200:
        print(json.dumps(response.json(), indent=4))
    else:
        print("Error:", response.status_code)

-- Using the JSON response from the #terminalSelection : Find a Setting with the Name "Sample Rate" and search for a "SupportedValue" with a description of "MSR Divide by 2" and substitute the "Settings" "Value" by the value of the "SupportedValue" "id".

-- # PUT the settings with /item/settings/ by using the ItemId as an params for the endpoint, and check the response

for module_id in module_ids:
    response = requests.get(url + "/item/settings/", params={"itemId": module_id})
    if response.status_code == 200:
        settings = response.json()
        for setting in settings["Settings"]:
            if setting["Name"] == "Sample Rate":
                for value in setting["SupportedValues"]:
                    if value["Description"] == "MSR Divide by 2":
                        setting["Value"] = value["Id"]
                        break
        response = requests.put(url + "/item/settings/", params={"itemId": module_id}, json=settings)
        if response.status_code != 200:
            print("Failed to set item settings")
    else:
        print("Error:", response.status_code)

# Change the Channels operation mode to ICP Input, set the voltage range to 1 V
-- # Change the "Operation Mode" of the Channels to a "SupportedValues" entry with "ICP" in the description
for channel_id in channel_ids:
    response = requests.get(url + "/item/operationMode/", params={"itemId": channel_id})
    if response.status_code == 200:
        operation_mode = response.json()
        for setting in operation_mode["Settings"]:
            for value in setting["SupportedValues"]:
                if "ICP" in value["Description"]:
                    setting["Value"] = value["Id"]
                    break
        response = requests.put(url + "/item/operationMode/", params={"itemId": channel_id}, json=operation_mode)
        if response.status_code != 200:
            print("Failed to set item operation mode")
    else:
        print("Error:", response.status_code)

# Update the Channel Voltage Range to 1 V
-- # Similar as above, change all channels /item/settings/ entry "Voltage Range" to "1 V"
for channel_id in channel_ids:
    response = requests.get(url + "/item/settings/", params={"itemId": channel_id})
    if response.status_code == 200:
        settings = response.json()
        for setting in settings["Settings"]:
            if setting["Name"] == "Voltage Range":
                for value in setting["SupportedValues"]:
                    if value["Description"] == "1 V":
                        setting["Value"] = value["Id"]
                        break
        response = requests.put(url + "/item/settings/", params={"itemId": channel_id}, json=settings)
        if response.status_code != 200:
            print("Failed to set item settings")
    else:
        print("Error:", response.status_code)

# Now apply the settings and show the mic is working
response = requests.put(url + "/system/settings/apply/")
if response.status_code == 200:
    print("Settings applied")
else:
    print("Failed to apply settings")

# Now stream the data
import socket
import struct

# First thing we need to do is to request which port is available for streaming.
# We can do this by requesting the /datastream/setup/ endpoint.
response = requests.get(url + "/datastream/setup/")
if response.status_code == 200:
    datastream_setup = response.json()
    tcp_port = datastream_setup["TCPPort"]
else:
    print("Error:", response.status_code)

try:
    tcp_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    tcp_socket.connect((ipAddress, tcp_port))
    print("TCP socket connected")
except Exception as e:
    print("Failed to open TCP socket:", str(e))

# Read the packet header
header_size = 32
header_data = tcp_socket.recv(header_size)

if len(header_data) == header_size:
    sequence_number = struct.unpack("<Q", header_data[:8])[0]
    transmit_timestamp = struct.unpack("<d", header_data[8:16])[0]
    buffer_level = struct.unpack("<f", header_data[16:20])[0]
    payload_size = struct.unpack("<I", header_data[20:24])[0]
    byte_order_marker = struct.unpack("<I", header_data[24:28])[0]
    payload_type = struct.unpack("<I", header_data[28:32])[0]
else:
    print("Failed to receive header data")

# Receive and decode the payload data
payload_data = tcp_socket.recv(payload_size)
while len(payload_data) < payload_size:
    payload_data += tcp_socket.recv(payload_size - len(payload_data))

received_payload_size = len(payload_data)
index = 0
analog_channel_data = {}

# Decode the generic channel header first
-- unpack the payload into local variables: 
channel_id = struct.unpack("<i", payload_data[index:index+4])[0]
sample_type = struct.unpack("<i", payload_data[index+4:index+8])[0]
channel_type = struct.unpack("<I", payload_data[index+8:index+12])[0]
channel_data_size = struct.unpack("<I", payload_data[index+12:index+16])[0]
timestamp_offset = struct.unpack("<Q", payload_data[index+16:index+24])[0]
index += 24

# Decode the specific channel header.
if channel_type == 0:
    channel_integrity = struct.unpack("<i", payload_data[index:index+4])[0]
    level_crossing_occurred = struct.unpack("<i", payload_data[index+4:index+8])[0]
    level = struct.unpack("<f", payload_data[index+8:index+12])[0]
    minimum = struct.unpack("<f", payload_data[index+12:index+16])[0]
    maximum = struct.unpack("<f", payload_data[index+16:index+20])[0]
    index += 20

    if channel_id not in analog_channel_data:
        analog_channel_data[channel_id] = []

    analog_channel_data[channel_id] += struct.unpack("f" * (channel_data_size // 4), payload_data[index:index+channel_data_size])
    index += channel_data_size

# Remember to read all the data
while index < received_payload_size:

# Close the socket
tcp_socket.close()

# Plot
from matplotlib import pyplot as plt

for channel_id, data in analog_channel_data.items():
    plt.plot(data, label=f"Channel {channel_id}")

plt.xlabel("Sample Index")
plt.ylabel("Channel Data")
plt.legend()
plt.show()