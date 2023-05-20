# Main.ps1
```powershell 
# ============================================
#                   Main.ps1
# ============================================

# Dit script zal de virtuele omgeving voor mijn BP automatisch opstellen. 
# Door het uitvoeren van dit script zou alles automatisch moeten worden opgestart en worden geconfigureerd. 
# De omgeving bevat volgende elementen:
    # 1. Een Windows Server 2019 die fungeert als AD-server
    # 2. Een Windows Client (Windows 10) die zal gebruikt worden om te verbinden met de AD. 
    # 3. Een Kali Linux VM die de aanvallen zal uitvoeren. Dit is de threat actor van dit project. 

[string] $PathToDCscript = "C:\Users\jornd\Desktop\Bachelorproef_DEMEYER_JORN\bachproef\environment\scripts\dc.ps1"
[string] $PathToClientScript = "C:\Users\jornd\Desktop\Bachelorproef_DEMEYER_JORN\bachproef\environment\scripts\client.ps1"
function log([string] $log)
{
    write-host "$log" -BackgroundColor White -ForegroundColor Red  
}

function CreateSharedFolders([string] $naam_folder) {
    if (Test-Path "C:\Users\jornd\Desktop\bp-shared\") {
        write-host "De map bestaat al, verdergaan..." -ForegroundColor Green 
    }
    else 
    {
        write-host "De map bestaat nog niet op het systeem. Deze wordt nu aangemaakt..." -ForegroundColor Magenta
        mkdir "C:\Users\jornd\Desktop\bp-shared\"
    }

    if (Test-Path "C:\Users\jornd\Desktop\bp-shared\$($naam_folder)") {
        write-host "De submap '$($naam_folder) bestaat al, verdergaan..." -ForegroundColor Green 
    }
    else {
        write-host "De submap '$($naam_folder)' bestaat nog niet, nu maken..." -ForegroundColor Magenta
        mkdir "C:\Users\jornd\Desktop\bp-shared\$($naam_folder)"
    }
}

function KaliVDI {
    if (Test-Path "C:\Users\jornd\Desktop\bp_environment\Kali Linux 2022.3 (64bit).vdi") {
        write-host 'De kali VDI staat al op zijn plaats en werd juist geconfigureerd!' -ForegroundColor Green 
        write-host 'Nu doorgaan naar de volgende stap...' -ForegroundColor Green 
    }
    else {
        write-host 'De kali VDI staat nog niet op zijn juiste plaats en wordt nu uitgepakt, even geduld...' -ForegroundColor Blue
        [string] $WinRARPath = 'C:\Program Files\WinRAR\WinRAR.exe'
        [string] $RARFile = 'C:\Users\jornd\Desktop\kali.7z'
        [string] $Destination = 'C:\Users\jornd\Desktop\'
        
        Start-Process $WinRARPath -ArgumentList "x $RARFile $Destination" -Wait 
        
        write-host 'Verplaatsen vdi naar de map bp_environment...' -ForegroundColor Blue 
        Move-Item "C:\Users\jornd\Desktop\64bit\Kali Linux 2022.3 (64bit).vdi" -Destination "C:\Users\jornd\Desktop\bp_environment\"
    
        write-host 'Aanpassen UUID van de vdi...' -ForegroundColor Blue 
        VBoxManage internalcommands sethduuid 'C:\Users\jornd\Desktop\bp_environment\Kali Linux 2022.3 (64bit).vdi'
    }    
}

 


# -----------------------------------
# 1. Opzetten DomeinController (dc)
# -----------------------------------
function MaakDc {
    # Aanmaken van de VM 
    log "Aanmaken van de domeincontroller"
    vboxmanage createvm --name "dc" --ostype "Windows2019_64" --register 
    
    # Aanmaken en toevoegen SATA controller 
    log "Aanmaken & toevoegen SATA controller..."
    vboxmanage storagectl "dc" --name "SATA Controller BP" --add sata 
    
    # Aanmaken en toevoegen IDE controller 
    log "Aanmaken & toevoegen IDE controller..."
    vboxmanage storagectl "dc" --name "IDE Controller BP" --add ide 
    
    # Aanmaken en toevoegen nieuwe harde schijf aan VM 
    log "Aamaken nieuwe harde schijf..."
    vboxmanage createmedium disk --filename "C:\Users\jornd\VirtualBox VMs\dc\dc.vdi" --size 30000 --format vdi 
    vboxmanage storageattach "dc" --storagectl "SATA Controller BP" --device 0 --port 0 --type hdd --medium "C:\Users\jornd\VirtualBox VMs\dc\dc.vdi"
    
    # Instellen van het aantal CPU's
    log "Instellen van het aanval CPU's..."
    vboxmanage modifyvm "dc" --cpus 2 
    
    # Instellen van het RAM en visuele geheugen 
    log "Instellen va het RAM en visuele geheugen"
    vboxmanage modifyvm "dc" --memory 2048
    vboxmanage modifyvm "dc" --vram 64 
    
    # Instellen netwerkadapaters
    log "Instellen netwerkadapters (NAT en internal network)..."
    vboxmanage modifyvm "dc" --nic1 nat 
    vboxmanage modifyvm "dc" --nic2 intnet 
    vboxmanage modifyvm "dc" --intnet2 "bp_intnet"
    
    # Koppelen ISO aan IDE controller
    log "Koppelen van de Windows Server 2019 ISO aan de IDE Controller..."
    VBoxManage storageattach "dc" --storagectl "IDE Controller BP" --port 0 --device 0 --type dvddrive --medium "C:\Users\jornd\Desktop\bp_environment\en_windows_server_2019_x64_dvd_4cb967d8.iso"
    
    
    # VM toevoegen aan een groep
    log "Aanmaken groep 'bachelorproef' en toevoegen van dc aan deze groep"
    if (Test-Path 'C:\Users\jornd\VirtualBox VMs\bachelorproef') {
        write-host "De groep 'bachelorproef' bestaat al, verdergaan..." -ForegroundColor Green  
    }
    else {
        vboxmanage modifyvm "dc" --groups "/bachelorproef"
    }
    
    # Aanmaken en configureren shared folders 
    log "Aanmaken en configureren van de shared folders..."
    CreateSharedFolders("dc-shared")
    
    log "Koppelen van de shared folder..."
    vboxmanage sharedfolder add "dc" --name "dc_shared" --hostpath "C:\Users\jornd\Desktop\bp-shared\dc-shared"
    
    # Inschakelen biderectioneel kopieëren en drag-&-drop
    log "Inschakelen biderectioneel kopieeren en drag-&-drop"
    vboxmanage modifyvm "dc" --clipboard bidirectional --draganddrop bidirectional 
    
    # Uitvoeren van een unattended install
    log "Start uitvoeren unattended install"
    vboxmanage unattended install "dc" `
    --iso="C:\Users\jornd\Desktop\bp_environment\en_windows_server_2019_x64_dvd_4cb967d8.iso" `
    --hostname=dc.bp-2223-jorn.hogent `
    --user=Administrator `
    --full-user-name=Administrator `
    --password "admin123" `
    --install-additions `
    --additions-iso="C:\Users\jornd\Desktop\bp_environment\VBoxGuestAdditions_6.1.36.iso" `
    --country "BE" `
    --start-vm="gui" `
    --post-install-command="powershell E:\vboxadditions\VBoxWindowsAdditions.exe /S ; Set-WinUserLanguageList -LanguageList fr-BE -Force ; timeout 20 ; shutdown /r" `
    --image-index=2 `

    # Time-out zodat virtuele machine opnieuw kan opstarten. 
    log "Kleine time-out zodat VM weer kan heropstarten..."
    Timeout /T 400 

    # Overzetten configuratie script
    vboxmanage guestcontrol "dc" copyto "C:\Users\jornd\Desktop\Bachelorproef_DEMEYER_JORN\bachproef\environment\scripts\dc.ps1" "C:\" --username "administrator" --password "admin123"
    vboxmanage guestcontrol "dc" copyto "C:\Users\jornd\Desktop\Bachelorproef_DEMEYER_JORN\bachproef\environment\scripts\vulnad.ps1" "C:\" --username "administrator" --password "admin123"
    vboxmanage guestcontrol "dc" copyto "C:\Users\jornd\Desktop\en_sql_server_2019_standard_x64_dvd_814b57aa.iso" "C:\" --username "administrator" --password "admin123"
    # Uitvoeren DC script
    #VBoxManage guestcontrol "dc" run --username "Administrator" --password "Test123!" --exe "C:\\windows\\system32\\WindowsPowerShell\v1.0\powershell.exe" -- powershell.exe /C "C:\dc.ps1" 

    }

# -----------------------------------
# 2. Opzetten Windows Client (client)
# -----------------------------------
function maakClient {
    # Aanmaken van de VM 
    log "Aanmaken van de domeincontroller"
    vboxmanage createvm --name "client" --ostype "Windows10_64" --register 
    
    # Aanmaken en toevoegen SATA controller 
    log "Aanmaken & toevoegen SATA controller..."
    vboxmanage storagectl "client" --name "SATA Controller BP" --add sata 
    
    # Aanmaken en toevoegen IDE controller 
    log "Aanmaken & toevoegen IDE controller..."
    vboxmanage storagectl "client" --name "IDE Controller BP" --add ide 
    
    # Aanmaken en toevoegen nieuwe harde schijf aan VM 
    log "Aamaken nieuwe harde schijf..."
    vboxmanage createmedium disk --filename "C:\Users\jornd\VirtualBox VMs\client\client.vdi" --size 30000 --format vdi 
    vboxmanage storageattach "client" --storagectl "SATA Controller BP" --device 0 --port 0 --type hdd --medium "C:\Users\jornd\VirtualBox VMs\client\client.vdi"
    
    # Instellen van het aantal CPU's
    log "Instellen van het aanval CPU's..."
    vboxmanage modifyvm "client" --cpus 2 
    
    # Instellen van het RAM en visuele geheugen 
    log "Instellen va het RAM en visuele geheugen"
    vboxmanage modifyvm "client" --memory 1536
    vboxmanage modifyvm "client" --vram 64 
    
    # Instellen netwerkadapaters
    log "Instellen netwerkadapters (NAT en internal network)..." 
    vboxmanage modifyvm "client" --nic1 intnet 
    vboxmanage modifyvm "client" --intnet1 "bp_intnet"
    
    # Koppelen ISO aan IDE controller
    log "Koppelen van de Windows Server 2019 ISO aan de IDE Controller..."
    VBoxManage storageattach "client" --storagectl "IDE Controller BP" --port 0 --device 0 --type dvddrive --medium "C:\Users\jornd\Desktop\bp_environment\en_windows_server_2019_x64_dvd_4cb967d8.iso"
    
    
    # VM toevoegen aan een groep
    log "Aanmaken groep 'bachelorproef' en toevoegen van client aan deze groep"
    if (Test-Path 'C:\Users\jornd\VirtualBox VMs\bachelorproef') {
        write-host "De groep 'bachelorproef' bestaat al, verdergaan..." -Foregrounclientolor Green  
    }
    else {
        vboxmanage modifyvm "client" --groups "/bachelorproef"
    }
    
    # Aanmaken en configureren shared folders 
    log "Aanmaken en configureren van de shared folders..."
    CreateSharedFolders("client-shared")
    
    log "Koppelen van de shared folder..."
    vboxmanage sharedfolder add "client" --name "client_shared" --hostpath "C:\Users\jornd\Desktop\bp-shared\client-shared"
    
    # Inschakelen biderectioneel kopieëren en drag-&-drop
    log "Inschakelen biderectioneel kopieeren en drag-&-drop"
    vboxmanage modifyvm "client" --clipboard bidirectional --draganddrop bidirectional 
    
    # Uitvoeren van een unattended install
    log "Start uitvoeren unattended install"
    vboxmanage unattended install "client" `
    --iso="C:\Users\jornd\Desktop\bp_environment\SW_DVD9_Win_Pro_10_20H2.10_64BIT_English_Pro_Ent_EDU_N_MLF_X22-76585.ISO" `
    --hostname=client.bp-2223-jorn.hogent `
    --user=Administrator `
    --full-user-name=Administrator `
    --password "Test123!" `
    --install-additions `
    --additions-iso="C:\Users\jornd\Desktop\bp_environment\VBoxGuestAdditions_6.1.36.iso" `
    --country "BE" `
    --start-vm="gui" `
    --post-install-command="powershell E:\vboxadditions\VBoxWindowsAdditions.exe /S ; Set-WinUserLanguageList -LanguageList fr-BE -Force ; timeout 20 ; shutdown /r" `
    --image-index=2 `

    Timeout /T 400 

    vboxmanage guestcontrol "client" copyto "C:\Users\jornd\Desktop\Bachelorproef_DEMEYER_JORN\bachproef\environment\scripts\client.ps1" "C:\client.ps1" --username "Administrator" --password "Test123!"
}

# -----------------------------------
# 3. Opzetten Kali Linux (kali)
# -----------------------------------
function maakKali {
    # Aanmaken van de virtuele machine
    log "Aanmaken van de eerste virtuele machine..."
    vboxmanage createvm --name "kali" --ostype "debian_64" --register 

    # Aanmaken en toevoegen SATA controller 
    log "Aanmaken & toevoegen SATA controller..."
    VBoxManage storagectl kali --name "SATA Controller BP" --add sata

    # Toevoegen van de VDI aan de SATA controller
    KaliVDI
    log 'Toevoegen van de kali VDI aan de SATA Controller...'
    vboxmanage storageattach kali --storagectl "SATA Controller BP" --device 0 --port 0 --type hdd --medium "C:\Users\jornd\Desktop\bp_environment\Kali Linux 2022.3 (64bit).vdi"

    # Instellen van het aantal CPUs 
    vboxmanage modifyvm "kali" --cpus 1 

    # Instellen van het RAM en Visuele geheugen 
    log "Instellen van het RAM-geheugen..."
    vboxmanage modifyvm "kali" --memory 1536
    vboxmanage modifyvm "kali" --vram 64

    # Wijzigen grafische controller
    log "Wijzigen grafische controller..."
    vboxmanage modifyvm "kali" --graphicscontroller vmsvga

    # Instellen netwerkadapters
    log "Instellen netwerkadapters"
    vboxmanage modifyvm "kali" --nic1 nat 
    vboxmanage modifyvm "kali" --nic2 intnet 
}

# Print een menu uit waarmee je kan kiezen welke VM je wenst te bouwen.
function PrintMenu() {    
    $antwoorden = New-Object System.Collections.ArrayList
    write-host '---------------MAIN.ps1---------------' -ForegroundColor Cyan
    write-host 'Dag gebruiker, wat wens je te doen?' -ForegroundColor Cyan 
    write-host 'Typ het woord in tussen de haken om je optie vast te leggen:' -ForegroundColor Cyan 
    write-host 'Alle VMs aanmaken? [all]' -ForegroundColor Blue
    write-host 'De DC aanmaken? [dc]' -ForegroundColor Blue
    write-host 'De Client aanmaken? [client]' -ForegroundColor Blue
    write-host 'De Kali VM aanmaken? [kali]' -ForegroundColor Blue
    write-host 'Stoppen? [end]' -ForegroundColor Red 

    [string] $invoer ='default'
    while ($invoer -ne 'end' -and $invoer -ne 'all') {
        $invoer = read-host 'Geef je optie(s) in'
        $antwoorden.Add($invoer.ToLower()) > $null 
    }
        if($antwoorden.IndexOf('all') -eq 0) {
            write-host 'all werd als eerste ingegeven en daarom stopte het programma meteen.' -ForegroundColor Yellow
        }
        else {
            $index_van_end = $antwoorden.IndexOf("end")
            $antwoorden.RemoveAt($index_van_end)
        }

        write-host 'Jouw antwoorden:' -ForegroundColor Green
        for ($i = 0; $i -lt $antwoorden.Count; $i++) {            
            Write-Host "$($i+1) -> $($antwoorden[$i])" -ForegroundColor Red 
        }  
    return $antwoorden 
}

$antwoorden = PrintMenu

function VoerKeuzeUit([string[]]$antwoorden) {
    if ($antwoorden.Length -eq 0) {
        write-host 'De array is leeg. Nu dit programma stoppen...' -ForegroundColor Red 
    }

    elseif ($antwoorden.Contains('all')) {
        write-host 'Er werd gekozen om alle VMs aan te maken, dit zal nu gebeuren...' -ForegroundColor Yellow
        maakDc 
        maakClient
        maakKali 
        break
    }
    else {
        if ($antwoorden.Contains('dc')) {
            write-host 'Er werd gekozen om de DC VM aan te maken. Dit zal nu gebeuren...' -ForegroundColor Yellow
            maakDc
        }

        if ($antwoorden.Contains('client')) {
            write-host 'Er werd gekozen om de Client VM aan te maken. Dit zal nu gebeuren...' -ForegroundColor Yellow        
            maakClient
        }

        if ($antwoorden.Contains('kali')) {
            write-host 'Er werd gekozen om de Kali VM aan te maken. Dit zal nu gebeuren...' -ForegroundColor Yellow
            maakKali
        }
    } 
}
VoerKeuzeUit $antwoorden
```