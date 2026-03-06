# MSI Utility v3 — 2026

**Erro ao executar no Windows e ajuste manual de MSI para controladores USB**

![Windows](https://img.shields.io/badge/Platform-Windows-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![PowerShell](https://img.shields.io/badge/Script-PowerShell-5391FE?style=for-the-badge&logo=powershell&logoColor=white)
![Registry](https://img.shields.io/badge/Method-Registry%20Edit-critical?style=for-the-badge)
![Use with caution](https://img.shields.io/badge/Status-Use%20with%20caution-orange?style=for-the-badge)

---

## Sobre

Este documento explica como lidar com falhas ao abrir o `MSI Utility v3` no Windows e como aplicar manualmente, via **Registro** ou **PowerShell**, ajustes relacionados a **Message Signaled Interrupts (MSI)** em controladores USB.

### Objetivos

- diagnosticar o erro de execução do utilitário;
- aplicar correções básicas;
- oferecer alternativa manual;
- documentar riscos e limitações técnicas.

> [!IMPORTANT]
> Este procedimento **não garante** ganho real de desempenho ou redução perceptível de input lag.  
> O efeito depende de hardware, driver, chipset, firmware e versão do Windows.

---

## Índice

- [Problema](#problema)
- [Causas prováveis](#causas-prováveis)
- [Soluções básicas](#soluções-básicas)
- [Ajuste manual pelo Registro](#ajuste-manual-pelo-registro)
- [Script PowerShell](#script-powershell)
- [O que o script faz](#o-que-o-script-faz)
- [Riscos e limitações](#riscos-e-limitações)
- [Rollback](#rollback)
- [Resumo rápido](#resumo-rápido)
- [Aviso final](#aviso-final)

---

## Problema

Ao tentar executar o `MSI Utility v3`, o Windows pode exibir mensagens como:

- **"O Windows não pode encontrar..."**
- **"Certifique-se de que o nome foi digitado corretamente e tente novamente."**
- falha ao abrir mesmo com **Executar como administrador**;
- o executável simplesmente não inicia.

---

## Causas prováveis

As causas mais comuns são:

1. **Arquivo bloqueado pelo Windows**
   - O sistema pode marcar arquivos baixados da internet como não confiáveis.

2. **Execução dentro de `.zip`**
   - O executável pode falhar se for aberto sem extração completa.

3. **Caminho problemático**
   - Pastas de usuário, caminhos longos ou locais sincronizados podem causar falhas.

4. **SmartScreen / Smart App Control**
   - O Windows pode bloquear binários não assinados ou com baixa reputação.

5. **Empacotamento incompleto**
   - Algumas cópias distribuídas por terceiros podem estar incompletas ou corrompidas.

---

## Soluções básicas

### 1) Desbloquear o arquivo

1. Clique com o botão direito em `MSI_util_v3.exe`
2. Abra **Propriedades**
3. Na aba **Geral**, procure por **Desbloquear**
4. Marque a opção
5. Clique em **Aplicar**
6. Execute novamente como administrador

---

### 2) Extrair o `.zip` antes de executar

Não execute o utilitário de dentro do arquivo compactado.

Faça assim:

```text
1. Extraia todo o conteúdo do .zip
2. Mova o executável para um caminho curto
3. Exemplo: C:\MSI\MSI_util_v3.exe
3) Evitar pastas problemáticas

Prefira caminhos como:

C:\MSI\
C:\Tools\MSI\

Evite:

Downloads
Área de Trabalho
Pastas sincronizadas com OneDrive
Pastas com nomes muito longos
Ajuste manual pelo Registro

Se o utilitário continuar falhando, o ajuste pode ser feito manualmente no Registro do Windows.

Pré-requisitos

conta com privilégios administrativos;

ponto de restauração criado;

backup/exportação das chaves que serão alteradas.

[!WARNING]
Alterar o Registro incorretamente pode causar instabilidade no sistema.

Passo 1 — localizar o dispositivo

Abra o Gerenciador de Dispositivos

Localize o Controlador Host USB

Clique em Propriedades

Vá à aba Detalhes

Em Propriedade, selecione Caminho da instância do dispositivo

Copie o valor exibido

Exemplo:

PCI\VEN_1234&DEV_5678&SUBSYS_00000000&REV_01\3&11583659&0&A0
Passo 2 — editar a chave de MSI

Abra o regedit e vá para:

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Enum\[INSTÂNCIA_DO_DISPOSITIVO]\Device Parameters\Interrupt Management\MessageSignaledInterruptProperties

Defina:

MSISupported = 1
Passo 3 — ajustar prioridade

Agora vá para:

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Enum\[INSTÂNCIA_DO_DISPOSITIVO]\Device Parameters\Interrupt Management\Affinity Policy

Crie ou ajuste:

DevicePriority = 3

Em muitos casos, o utilitário gráfico faz exatamente esse tipo de alteração.
A interface muda; a engrenagem subterrânea continua sendo o Registro.

Script PowerShell

O script abaixo percorre controladores USB presentes no sistema e tenta aplicar:

MSISupported = 1

DevicePriority = 3

Execute como Administrador
# Busca controladores USB presentes no sistema
$usbControllers = Get-PnpDevice -PresentOnly | Where-Object {
    $_.FriendlyName -like "*USB Host Controller*" -or
    $_.FriendlyName -like "*USB 3.0*" -or
    $_.FriendlyName -like "*USB 3.1*"
}

foreach ($dev in $usbControllers) {
    $instanceId = $dev.InstanceId
    $registryPath = "HKLM:\SYSTEM\CurrentControlSet\Enum\$instanceId\Device Parameters\Interrupt Management"

    # Habilita MSI, se a chave existir
    if (Test-Path "$registryPath\MessageSignaledInterruptProperties") {
        Set-ItemProperty -Path "$registryPath\MessageSignaledInterruptProperties" -Name "MSISupported" -Value 1 -Force
        Write-Host "MSI ativado para: $($dev.FriendlyName)" -ForegroundColor Cyan
    }
    else {
        Write-Host "Chave MSI não encontrada para: $($dev.FriendlyName)" -ForegroundColor DarkYellow
    }

    # Cria Affinity Policy se necessário
    if (-not (Test-Path "$registryPath\Affinity Policy")) {
        New-Item -Path "$registryPath\Affinity Policy" -Force | Out-Null
    }

    # Define prioridade
    Set-ItemProperty -Path "$registryPath\Affinity Policy" -Name "DevicePriority" -Value 3 -Force
    Write-Host "Prioridade aplicada para: $($dev.FriendlyName)" -ForegroundColor Green
}

Write-Host "`nConcluído. Reinicie o PC para aplicar as alterações." -ForegroundColor Yellow
O que o script faz
MSISupported = 1

Tenta habilitar o uso de Message Signaled Interrupts (MSI) no dispositivo, quando o driver e o hardware suportam isso.

DevicePriority = 3

Define uma prioridade para a política de interrupção do dispositivo.

Riscos e limitações

Antes de aplicar esse ajuste, considere o seguinte:

nem todo dispositivo suporta MSI corretamente;

nem todo driver reage bem à mudança;

o efeito pode ser nulo;

o sistema pode ficar instável em alguns cenários;

não há garantia de redução de input lag;

aplicar em massa sem validação é uma forma elegante de cutucar o kernel com uma vara.

Recomendações

crie um ponto de restauração;

exporte as chaves antes de editar;

altere poucos dispositivos por vez;

reinicie o sistema;

valide estabilidade antes de manter a mudança.

Rollback

Se quiser desfazer manualmente, volte os valores para o padrão anterior.

Opção 1 — Registro

Defina:

MSISupported = 0

E remova ou ajuste:

DevicePriority
Opção 2 — PowerShell
$usbControllers = Get-PnpDevice -PresentOnly | Where-Object {
    $_.FriendlyName -like "*USB Host Controller*" -or
    $_.FriendlyName -like "*USB 3.0*" -or
    $_.FriendlyName -like "*USB 3.1*"
}

foreach ($dev in $usbControllers) {
    $instanceId = $dev.InstanceId
    $registryPath = "HKLM:\SYSTEM\CurrentControlSet\Enum\$instanceId\Device Parameters\Interrupt Management"

    if (Test-Path "$registryPath\MessageSignaledInterruptProperties") {
        Set-ItemProperty -Path "$registryPath\MessageSignaledInterruptProperties" -Name "MSISupported" -Value 0 -Force
        Write-Host "MSI desativado para: $($dev.FriendlyName)" -ForegroundColor Yellow
    }

    if (Test-Path "$registryPath\Affinity Policy") {
        Remove-ItemProperty -Path "$registryPath\Affinity Policy" -Name "DevicePriority" -ErrorAction SilentlyContinue
        Write-Host "DevicePriority removido para: $($dev.FriendlyName)" -ForegroundColor DarkYellow
    }
}

Write-Host "`nRollback concluído. Reinicie o PC." -ForegroundColor Cyan
Resumo rápido
Situação	Ação
Windows não encontra o executável	Extraia o .zip e mova para um caminho curto
Executável não abre	Desbloqueie nas propriedades
Continua falhando	Tente executar como administrador
Ainda não abre	Faça o ajuste manual no Registro
Quer automatizar	Use o script PowerShell
Deu ruim	Faça rollback e reinicie
Aviso final

O MSI Utility v3 é, na prática, uma interface para ajustes de interrupção em dispositivos suportados.
Se ele não abrir, o ajuste manual pode funcionar como alternativa — mas deve ser aplicado com critério, backup e teste.

Este material não promete ganho garantido de performance. Ele documenta um procedimento técnico que pode ajudar em alguns casos e não fazer diferença alguma em outros.

O Windows, esse bicho estranho, às vezes coopera; às vezes faz teatro.
