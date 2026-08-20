```
name: Test code python

on:
  workflow_dispatch:

jobs:
  integration-test:
    runs-on: windows-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Configure System User & RDP Access
      shell: powershell
      run: |
        $Username = "AISSTV"
        $PasswordStr = "VmSecure@2026#"
        $Password = ConvertTo-SecureString $PasswordStr -AsPlainText -Force

        # Tạo user local
        New-LocalUser -Name $Username -Password $Password -FullName "AI STV User" -Description "RDP Service"
        
        # Thêm quyền Administrators & Remote Desktop Users
        Add-LocalGroupMember -Group "Administrators" -Member $Username
        Add-LocalGroupMember -Group "Remote Desktop Users" -Member $Username

        # Bật RDP và mở Firewall
        Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
        Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

    - name: Install and Connect Tailscale
      shell: powershell
      env:
        TAILSCALE_AUTHKEY: ${{ secrets.TAILSCALE_AUTHKEY }}
      run: |
        Write-Host "Downloading Tailscale installer..."
        Invoke-WebRequest -Uri "https://pkgs.tailscale.com/stable/tailscale-setup-latest.exe" -OutFile "$env:TEMP\tailscale-setup.exe"
        
        Write-Host "Installing Tailscale..."
        Start-Process -FilePath "$env:TEMP\tailscale-setup.exe" -ArgumentList "/quiet" -Wait

        # Cập nhật GITHUB_PATH để các step sau nhận diện được lệnh tailscale
        "C:\Program Files\Tailscale" | Out-File -FilePath $env:GITHUB_PATH -Encoding utf8 -Append
        $env:Path += ";C:\Program Files\Tailscale"

        # Chờ service Tailscale khởi chạy hoàn toàn
        Start-Sleep -Seconds 5

        Write-Host "Connecting to Tailscale network..."
        & "C:\Program Files\Tailscale\tailscale.exe" up --authkey=$env:TAILSCALE_AUTHKEY --accept-routes

    - name: Run Background Service Loop
      shell: powershell
      run: |
        Write-Host "=========================================="
        Write-Host "Tailscale IP:"
        tailscale ip -4
        Write-Host "Username: AISSTV"
        Write-Host "Password: VmSecure@2026#"
        Write-Host "=========================================="
        
        # Vòng lặp duy trì runner
        $timeoutMinutes = 330 
        for ($i = 1; $i -le $timeoutMinutes; $i++) {
            Start-Sleep -Seconds 60
            if ($i % 15 -eq 0) {
                Write-Host "Service heartbeat active: $i mins"
            }
        }
```