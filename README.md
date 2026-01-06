
# android-apk-protection-signing-playbook


![Platform: Android](https://img.shields.io/badge/platform-Android-green)
![Status: Educational](https://img.shields.io/badge/status-Educational-blue)


> **Resumo**: Este repositório demonstra, de forma prática e segura, o processo técnico de **proteção** (via ferramenta de RASP), **alinhamento (zipalign)**, **assinatura (apksigner)**, **verificação** e **instalação** de APKs Android para **testes em emulador**.  
> **Importante**: Todo o conteúdo é **genérico** e não expõe dados confidenciais, credenciais, licenças ou artefatos proprietários.


## Sumário
- [Contexto & Objetivo](#1-contexto--objetivo)
- [Ambiente & Ferramentas](#2-ambiente--ferramentas)
- [Pipeline / Passo a passo](#3-pipeline--passo-a-passo)
- [Assinatura & parâmetros](#4-assinatura--parametros)
- [Boas práticas](#5-boas-práticas)
- [Erros comuns & soluções](#6-erros-comuns--soluções)
- [Validação](#7-validação)
- [Referências](#8-referências)


---

## 1) Contexto & Objetivo

- **Objetivo**: Demonstrar o pipeline técnico para proteger um APK com uma ferramenta de RASP, alinhar, assinar, verificar e instalar em emulador.
- **Escopo**: *Pipeline técnico* (foco em operação das ferramentas; não cobre políticas, arquitetura ou governança).
- **App de exemplo**: `SimpleSimon`.  
  > Para o teste, o app precisa da **permissão de internet** para acionar o fluxo de tamper que abre uma URL (conforme blueprint) quando a execução falhar em condições específicas (ex.: emulador).
- **Portfólio**: Este conteúdo é ideal como **portfólio pessoal** em **AppSec / DevSecOps / Mobile Security**.
- **Ferramenta de proteção**: **ferramenta de RASP** (Runtime Application Self-Protection), com binário para proteção de APKs no Windows.  

---

## 2) Ambiente & Ferramentas

- **OS**: Windows 11
- **Android SDK Location**: `C:\Users\MeuUser\AppData\Local\Android\Sdk`
- **Build-tools**: `36.1.0` 
- **Ferramentas**:
  - `zipalign` (Android Build Tools)
  - `apksigner` (Android Build Tools)
  - `keytool` (JDK 25, ex.: Zulu; caminho típico: `C:\Program Files\Java\Zulu\zulu-25\bin`)
  - `adb` (Android Platform-Tools)
  - **Emulador Android (AVD)** — API Level conforme sua configuração
  - **Ferramenta de RASP** com binário para proteção de APKs no Windows (ex.: `protect-android.exe`) — **não versionar**
  - **Blueprint JSON** (exemplo genérico abaixo; sem segredos)

**Caminhos de exemplo (genéricos):**
- APK protegido/alinhado:  
  `C:\Users\MeuUser\Documents\RASP\apps\protected\SimpleSimon\SimpleSimon-aligned.apk`
- Keystore:  
  `C:\Users\MeuUser\Documents\RASP\Android\protect-android-windows\bin\meuapp.keystore`
- Saída do APK assinado:  
  `C:\Users\MeuUser\Documents\RASP\apps\assinados\SimpleSimon-assinado.apk`

---

## 3) Pipeline / Passo a passo

### 3.1 Proteção (opcional, com blueprint)

Este projeto demonstra proteção via **ferramenta de RASP** com **blueprint (JSON)** que define **guards** e **tamper action**.

**Guards usados (exemplo):**
- `emulatorDetection` — detecta execução em **emulador**
- `hookDetection` — detecta frameworks de **hook/instrumentação**
- `rootDetection` — detecta **root** no dispositivo

**Tamper Action (exemplo):**
- `type: openUrlAndFail` — abre uma URL informativa e **bloqueia a execução do app**.
- `immediateAction: false` — aciona a ação **no fluxo de inicialização**, não imediatamente.




**Exemplo de blueprint (genérico)** — `blueprints/example-protection.json`:
```json
{
  "appAware": true,
  "guards": [
    "emulatorDetection",
    "hookDetection",
    "rootDetection"
  ],
  "tamperAction": {
    "immediateAction": false,
    "type": "openUrlAndFail",
    "url": "https://example.com/security-help"
  }
}
```

## Comando de proteção (genérico / vendor-agnostic):
```bat
@echo off
set RASP_PROTECT_BIN=C:\caminho\para\binarios\protect-android.exe
set BLUEPRINT=C:\caminho\para\blueprints\example-protection.json
set INPUT_APK=C:\caminho\para\AppOriginal.apk
set OUTPUT_DIR=C:\caminho\para\output

"%RASP_PROTECT_BIN%" ^
  -b "%BLUEPRINT%" ^
  -i "%INPUT_APK%" ^
  -o "%OUTPUT_DIR%" ^
  --verbose

if %ERRORLEVEL% NEQ 0 (
  echo [ERRO] Protecao falhou. Verifique binario, licenca e blueprint.
  exit /b 1
)

echo [OK] Protecao concluida. APK gerado normalmente como unsigned/unaligned.

```

### 3.2 Gerar keystore (se ainda não tiver)
```bat

"C:\Program Files\Java\Zulu\zulu-25\bin\keytool" -genkeypair ^
  -alias meuapp ^
  -keyalg RSA -keysize 2048 ^
  -validity 3650 ^
  -keystore "C:\caminho\para\meuapp.keystore"
```

### 3.3 Zipalign (sempre antes de assinar)
```bat

"C:\Users\MeuUser\AppData\Local\Android\Sdk\build-tools\36.1.0\zipalign.exe" -v -p 4 ^
  "C:\Users\MeuUser\Documents\RASP\apps\protected\SimpleSimon\SimpleSimon-unaligned-unsigned-protected.apk" ^
  "C:\Users\MeuUser\Documents\RASP\apps\protected\SimpleSimon\SimpleSimon-aligned.apk"
```
### 3.4 Assinatura (apksigner — APK Signature v2/v3)
```bat

"C:\Users\MeuUser\AppData\Local\Android\Sdk\build-tools\36.1.0\apksigner.bat" sign ^
  --ks "C:\Users\MeuUser\Documents\RASP\Android\protect-android-windows\bin\meuapp.keystore" ^
  --ks-key-alias meuapp ^
  --ks-pass pass:%KS_PASS% ^
  --key-pass pass:%KEY_PASS% ^
  --out "C:\Users\MeuUser\Documents\RASP\apps\assinados\SimpleSimon-assinado.apk" ^
  "C:\Users\MeuUser\Documents\RASP\apps\protected\SimpleSimon\SimpleSimon-aligned.apk"
```
### 3.5 Verificação da assinatura
```bat

"C:\Users\MeuUser\AppData\Local\Android\Sdk\build-tools\36.1.0\apksigner.bat" verify --verbose --print-certs ^
  "C:\Users\MeuUser\Documents\RASP\apps\assinados\SimpleSimon-assinado.apk"
```
### 3.6 Instalação no emulador
    Dica: Desinstale versões anteriores se o package name for igual e a assinatura diferente (para evitar INSTALL_FAILED_UPDATE_INCOMPATIBLE).
```bat

"C:\Users\MeuUser\AppData\Local\Android\Sdk\platform-tools\adb.exe" install -r ^
  "C:\Users\MeuUser\Documents\RASP\apps\assinados\SimpleSimon-assinado.apk"
```
  Desinstalar (se necessário):
```bat
  "C:\Users\MeuUser\AppData\Local\Android\Sdk\platform-tools\adb.exe" uninstall com.exemplo.simplesimon
```


## 4) Assinatura & parâmetros

Alias: meuapp  
Validade: 10 anos (3650 dias)  
Algoritmo: RSA 2048  
DN genérico: CN=User, OU=MobileSecurity, O=ExampleOrg, L=City, ST=State, C=BR


## 5) Boas práticas

- Ordem correta: proteger → zipalign → assinar → verificar → testar.
- Nunca faça zipalign depois de assinar (invalida a assinatura).
- Não publique APKs reais, keystores ou senhas.
- Use variáveis de ambiente para senhas (KS_PASS, KEY_PASS).
- Documente o package name e o API Level do AVD para reprodutibilidade.

## 6) Erros comuns & soluções

- failed to start adb
Verifique instalação do Platform-Tools, PATH correto e reinicie o adb:
```bat
adb kill-server
adb start-server
adb devices
```
- INSTALL_FAILED_UPDATE_INCOMPATIBLE
Desinstale a versão anterior (mesmo package com assinatura diferente):
```bat
adb uninstall com.exemplo.simplesimon
```
- Warnings do apksigner com Java (métodos restritos)
São avisos de módulos nas versões recentes do Java; não impactam a assinatura.

## 7) Validação

- Assinatura:
```bat
apksigner.bat verify --verbose --print-certs "C:\...\SimpleSimon-assinado.apk"
```
- Emulador:
```bat
adb.exe devices
adb.exe install -r "C:\...\SimpleSimon-assinado.apk"
adb.exe shell pm list packages | findstr simon
```

## 8) Referências

https://developer.android.com/studio/publish/app-signing
https://developer.android.com/studio/command-line/zipalign
https://developer.android.com/studio/command-line/apksigner


> **Aviso legal**  
> Repositório com fins educacionais e de portfólio pessoal. Não inclui apps oficiais nem informações sensíveis. Respeite licenças e termos das ferramentas de RASP e do Android SDK.
