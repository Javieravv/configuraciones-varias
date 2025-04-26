# Configurar terminal de windows 11.

1. Instalar las siguientes aplicaciones de la tienda de apps de Windows 11:

- Windows Terminal.
- PowerShell
- WigGet

2. Entrar a configuración y escoger por defecto:
Perfil predeterminado: PowerShell
Aplicación de terminal predeterminada: Terminal de windows.

3. Escoger los colores que se quieran o crear una nueva paleta de colores al gusto.

4. Instalarle un tema para que el prompt se vea diferente.

Ir a la web https://ohmyposh.dev/
Ahí pueden escogerse múltiples temas.

Lo instalamos con este comando: 
```
winget install JanDeDobbeleer.OhMyPosh -s winget
```
5. Instalar fuentes personalizadas.

Abrimos la terminal con usuario de administrador y ejecutamos:
```
oh-my-posh font install
```

Escoger la fuente que se quiera. Se recomienda FiraCode

6. Configurar la fuente en la terminal.

En valores predeterminados -> Apariencia -> fuentes

Escoger la fuente FiraCode Nerd Mono

7. Activamos Oh-My-Posh dentro de la terminal.

```
oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH\jandedobbeleer.omp.json"
```
Este comando mostrará este otro comando:

```
(@(& 'C:/Users/javie/AppData/Local/Programs/oh-my-posh/bin/oh-my-posh.exe' init pwsh --config='C:\Users\javie\AppData\Local\Programs\oh-my-posh\themes\jandedobbeleer.omp.json' --print) -join "`n") | Invoke-Expression
```
Lo copiamos y pegamos. Aparecerá la configuracion escogida, la cual no se mantendrá.
Para mantenerla debemos este comando colocarlo en otro lugar.

En la web de Oh-my-Posh escoger la opción de DefaultThems 

8. Para escoger otro tema.

Escribir el comando 

```
Get-PoshThemes
```

Escogemos el tema que guste. Por ejemplo pure. Buscamos el tema y manteniendo el control oprimido damos click sobre su nombre para que se abra su archivo de configuración. Así tomaremos el nombre de ese archivo que se usará más adelante.

El nombre del archivo lo colocamos en la siguiente línea:
```
(@(& 'C:/Users/javie/AppData/Local/Programs/oh-my-posh/bin/oh-my-posh.exe' init pwsh --config='C:\Users\javie\AppData\Local\Programs\oh-my-posh\themes\pure.omp.json' --print) -join "`n") | Invoke-Expression
```

9. Ahora abriremos el archivo que contiene las configuraciones de la terminal por defecto.

```
notepad $PROFILE 
```
Si aparece error, ejecutar este comando y luego de nuevo el comando anterior:

```
New-Item -Path $PROFILE -Type File -Force 
```

En el nuevo archivo que se abre con NOTEPAD, pegar este comando:
```
(@(& 'C:/Users/javie/AppData/Local/Programs/oh-my-posh/bin/oh-my-posh.exe' init pwsh --config='C:\Users\javie\AppData\Local\Programs\oh-my-posh\themes\pure.omp.json' --print) -join "`n") | Invoke-Expression
```
Este contiene la configuración del tema.

10. Agregamos temas para ver archivos, carpetas.

Con este comando:

```
Install-Module -Name Terminal-Icons -Repository PSGallery
```
Se deben activar:
```
Import-Module Terminal-Icons 
```

Esto se debe incluir en el archivo de $PROFILE 

11. Ahora instalamos téxto predictivo con el módulo PSReadLine.

```
Set-PSReadLineOption -PredictionViewStyle ListView
```

Agregamos esto también al perfil.

