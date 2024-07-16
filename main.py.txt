import subprocess
import re
import platform
import winreg as reg

def get_interface():
    try:
        if platform.system() == "Windows":
            output = subprocess.check_output("wmic nic where NetEnabled=true get NetConnectionID, Index", shell=True).decode()
            interfaces = output.split('\n')[1:-1]
            return [tuple(iface.strip().rsplit(None, 1)) for iface in interfaces if iface.strip()]
        elif platform.system() == "Linux":
            output = subprocess.check_output("ip link show", shell=True).decode()
            return re.findall(r'^\d+: (\w+):', output, re.MULTILINE)
        else:
            print("Unsupported operating system.")
            return []
    except subprocess.CalledProcessError as e:
        print(f"Error listing interfaces: {e}")
        return []

def change_mac_address(interface, new_mac):
    try:
        if platform.system() == "Windows":
            interface_name, index = interface
            key_path = fr"SYSTEM\CurrentControlSet\Control\Class\{{4D36E972-E325-11CE-BFC1-08002bE10318}}\{index.zfill(4)}"
            with reg.OpenKey(reg.HKEY_LOCAL_MACHINE, key_path, 0, reg.KEY_WRITE) as key:
                reg.SetValueEx(key, "NetworkAddress", 0, reg.REG_SZ, new_mac.replace(':', ''))
                print(f"MAC address for {interface_name} changed to {new_mac}. Please restart the network interface.")
        elif platform.system() == "Linux":
            subprocess.run(f"sudo ip link set dev {interface} down", shell=True, check=True)
            subprocess.run(f"sudo ip link set dev {interface} address {new_mac}", shell=True, check=True)
            subprocess.run(f"sudo ip link set dev {interface} up", shell=True, check=True)
            print(f"MAC address for {interface} changed to {new_mac}")
    except (subprocess.CalledProcessError, PermissionError, reg.WindowsError) as e:
        print(f"Error changing MAC address: {e}")

def validate_mac(mac):
    return bool(re.match(r'^([0-9A-Fa-f]{2}[:-]){5}([0-9A-Fa-f]{2})$', mac))

def main():
    interfaces = get_interface()
    if not interfaces:
        return

    print("Available Network Interfaces:")
    for i, iface in enumerate(interfaces):
        print(f"{i}: {iface[0]}")

    try:
        choice = int(input("Select the interface number: "))
        if choice < 0 or choice >= len(interfaces):
            print("Invalid choice.")
            return
    except ValueError:
        print("Invalid input. Please enter a number.")
        return

    new_mac = input("Enter new MAC address (format: XX:XX:XX:XX:XX:XX): ")
    if not validate_mac(new_mac):
        print("Invalid MAC address format.")
        return

    change_mac_address(interfaces[choice], new_mac)

if __name__ == "__main__":
    main()
