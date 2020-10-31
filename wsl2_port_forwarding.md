## WSL2 configuration to get localhost access on Windows
1. Create an environment variable for the **adapter** ip address. This will always give you Windows' ip address which you can reference in your apps. Add the following to your `.bashrc` so you don't have to run this each time.
`export WSL_HOST_IP=$(grep -Po '(?<=nameserver ).*' /etc/resolv.conf)`


2. Windows will listen on 0.0.0.0, but not on localhost/127.0.0.1. You need to set up forwarding from 0.0.0.0 to localhost on the Windows side. This can be done with the following command from cmd.exe (you will need admin rights):
`netsh interface portproxy add v4tov4 listenport=<your_port> listenaddress=0.0.0.0 connectport=<your_port> connectaddress=127.0.0.1`

3. You will need to create a persistent firewall rule for Windows to allow inbound connections from WSL2. Run the following in elevated-access PowerShell:
`New-NetFirewallRule -DisplayName "WSL" -Direction Inbound  -InterfaceAlias "vEthernet (WSL)"  -Action Allow`