@echo off
setlocal EnableExtensions EnableDelayedExpansion

title Manutencao Automatica do Windows
color 0A

:: ===============================
:: Verifica se está como Administrador
:: ===============================
net session >nul 2>&1
if %errorlevel% neq 0 (
    echo.
    echo ======================================
    echo Execute este arquivo como Administrador.
    echo ======================================
    pause
    exit
)

:: ===============================
:: Cria pasta de logs
:: ===============================
if not exist "C:\Logs" mkdir "C:\Logs"

set LOG=C:\Logs\Manutencao.log

echo.>>"%LOG%"
echo ================================================>>"%LOG%"
echo %date% %time% - Inicio da manutencao>>"%LOG%"
echo ================================================>>"%LOG%"

:: ===============================
:: Caminho deste próprio arquivo
:: ===============================
set SCRIPT=%~f0

:: ===============================
:: Cria/Atualiza tarefa agendada
:: ===============================
schtasks /Query /TN "Manutencao Windows Automatica" >nul 2>&1

if errorlevel 1 (

    echo Criando tarefa agendada...

    schtasks /Create ^
    /TN "Manutencao Windows Automatica" ^
    /TR "\"%SCRIPT%\"" ^
    /SC HOURLY ^
    /MO 2 ^
    /RL HIGHEST ^
    /RU SYSTEM ^
    /F

    echo %date% %time% - Tarefa criada>>"%LOG%"

) else (

    schtasks /Change ^
    /TN "Manutencao Windows Automatica" ^
    /TR "\"%SCRIPT%\"" >nul 2>&1

)

:: ====================================================
:: [AUTOMÁTICO] Geração de Nome Aleatório para o PC
:: ====================================================
echo Gerando novo nome aleatorio para o computador...
set "CHARS=ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
set "NOVO_NOME=PC-"
for /L %%i in (1,1,6) do (
    set /a "RAND=!random! %% 36"
    for %%j in (!RAND!) do set "NOVO_NOME=!NOVO_NOME!!CHARS:~%%j,1!"
)

echo Alterando nome do sistema para !NOVO_NOME!...
wmic computersystem where name="%COMPUTERNAME%" call rename name="!NOVO_NOME!" >>"%LOG%" 2>&1
echo %date% %time% - Nome do computador alterado para !NOVO_NOME! (Aplica no proximo reboot)>>"%LOG%"


:: ====================================================
:: [AUTOMÁTICO] Alteração Aleatória do Endereço MAC
:: ====================================================
echo Gerando e alterando endereco MAC da placa de rede...
powershell -NoProfile -ExecutionPolicy Bypass -Command "^
    $adapter = Get-NetAdapter | Where-Object Status -eq 'Up' | Select-Object -First 1; ^
    if ($adapter) { ^
        $hex = '0','1','2','3','4','5','6','7','8','9','A','B','C','D','E','F'; ^
        $laa = '2','6','A','E' | Get-Random; ^
        $rest = (1..10 | ForEach-Object { $hex | Get-Random }) -join ''; ^
        $mac = '0' + $laa + $rest; ^
        Set-NetAdapter -Name $adapter.Name -MacAddress $mac -Confirm:$false; ^
        Write-Output $mac; ^
    }" >>"%LOG%" 2>&1


echo.
echo ===============================
echo Iniciando manutencao de Rede...
echo ===============================

echo Limpando DNS...
ipconfig /flushdns >>"%LOG%"

echo Limpando cache ARP...
arp -d * >>"%LOG%" 2>&1

:: ====================================================
:: [AUTOMÁTICO] Renovação e Força de Troca de IP DHCP
:: ====================================================
echo Liberando IP atual (DHCP Release)...
ipconfig /release >>"%LOG%"

echo Reiniciando adaptadores de rede para aplicar novos dados...
powershell -NoProfile -Command "Get-NetAdapter | Where-Object Status -eq 'Up' | Restart-NetAdapter" >nul 2>&1

echo Solicitando novo IP ao servidor DHCP (Renew)...
ipconfig /renew >>"%LOG%"

echo Reiniciando Winsock...
netsh winsock reset >>"%LOG%"

echo Reiniciando TCP/IP...
netsh int ip reset >>"%LOG%"

echo.
echo ===============================
echo Iniciando Limpeza do Sistema...
echo ===============================

echo Limpando TEMP...
del /f /s /q "%TEMP%\*" >nul 2>&1
for /d %%i in ("%TEMP%\*") do rd /s /q "%%i" >nul 2>&1

echo Limpando Windows Temp...
del /f /s /q "C:\Windows\Temp\*" >nul 2>&1
for /d %%i in ("C:\Windows\Temp\*") do rd /s /q "%%i" >nul 2>&1

echo Limpando Prefetch...
del /f /s /q "C:\Windows\Prefetch\*" >nul 2>&1

echo Limpando cache de miniaturas...
del /f /q "%LOCALAPPDATA%\Microsoft\Windows\Explorer\thumbcache*" >nul 2>&1

echo Limpando Google Chrome...
taskkill /IM chrome.exe /F >nul 2>&1
rmdir /s /q "%LOCALAPPDATA%\Google\Chrome\User Data\Default\Cache" 2>nul
rmdir /s /q "%LOCALAPPDATA%\Google\Chrome\User Data\Default\Code Cache" 2>nul
rmdir /s /q "%LOCALAPPDATA%\Google\Chrome\User Data\Default\GPUCache" 2>nul

echo Limpando Microsoft Edge...
taskkill /IM msedge.exe /F >nul 2>&1
rmdir /s /q "%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\Cache" 2>nul
rmdir /s /q "%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\Code Cache" 2>nul
rmdir /s /q "%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\GPUCache" 2>nul

echo Limpando Firefox...
taskkill /IM firefox.exe /F >nul 2>&1
for /d %%i in ("%LOCALAPPDATA%\Mozilla\Firefox\Profiles\*") do (
    rmdir /s /q "%%i\cache2" 2>nul
)

echo Limpando Lixeira...
PowerShell -NoProfile -Command "Clear-RecycleBin -Force" >nul 2>&1

echo Verificando arquivos do Windows...
sfc /scannow >>"%LOG%"

echo Executando DISM...
DISM /Online /Cleanup-Image /RestoreHealth >>"%LOG%"

echo %date% %time% - Manutencao finalizada>>"%LOG%"

echo.
echo ======================================
echo MANUTENCAO CONCLUIDA
echo.
echo Novo Nome Gerado: !NOVO_NOME! (Aplica no proximo reboot)
echo A tarefa automatica esta instalada (a cada 2h).
echo Log: C:\Logs\Manutencao.log
echo ======================================

pause
