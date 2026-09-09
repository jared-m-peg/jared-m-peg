Building RSC+ using its source code on Windows 10

1\. Download the prerequisites (Git, Apache Ant, and JDK 1.8) for building RSC+

Download and install Git from here: `https://git-scm.com/install/windows`

Before installing it, check that its digital signature is OK:
Right click the following file: `Git-2.55.0.5-64-bit.exe`
Click on `Properties`
Click on the `Digital Signatures` tab
Click on the listed signature
Click on `Details`
It should print: `The digital signature is OK.`

Download the Apache Ant binary, the ASC file, and the KEYS file:
Go to: `https://downloads.apache.org/ant/`
Download the following file: `KEYS`
Go to: `https://downloads.apache.org/ant/binaries/`
Download these two files:
```text
apache-ant-1.10.17-bin.zip
apache-ant-1.10.17-bin.zip.asc
```
Place all three files in the same folder.

We need to install Gpg4win so we can check that the Apache Ant zip file has a valid signature from the KEYS file

Download Gpg4win from here: `https://gpg4win.org/index.html`

Right-click the file, click Properties, click the Digital Signatures tab, click on the listed signature, click Details, and check whether it says, "The digital signature is OK."

Open a Command Prompt window (cmd.exe) and use this command to import the KEYS file:
```cmd
gpg --import C:\Users\standard_2\Downloads\BuildingSourceCode\ApacheAnt\KEYS
```
In the same cmd.exe window, change to the directory where the three Apache Ant files are located:
```cmd
cd C:\Users\standard_2\Downloads\BuildingSourceCode\ApacheAnt
```
Use this command to verify the zip file:
```cmd
gpg --verify apache-ant-1.10.17-bin.zip.asc apache-ant-1.10.17-bin.zip
```
It should print:
```text
gpg: Good signature from "Stefan Bodewig <bodewig@apache.org>"
```
And the signing key:
6A93161EB1990E8346E7BA2B23738DFD7C40DE43
The GPG output shows that the Apache Ant zip file has a valid signature from Stefan Bodewig's key, 6A93161EB1990E8346E7BA2B23738DFD7C40DE43

Right-click the `apache-ant-1.10.17-bin.zip` folder
Click on `Extract All...`
Navigate to `C:\Tools`
Then click on `Extract`

Download JDK 8 from here: `https://adoptium.net/temurin/releases/?version=8&os=any&arch=any`
Under Windows, select JDK and x64, and then right-click MSI and open in new tab to download the file
On this same page click Checksum to see the sha256 checksum
Go to Windows Explorer, go to the folder where downloaded file is located, right click on the space, and click on Open in Terminal to open a Powershell window
In the Powershell window use this command:
```text
certUtil -hashfile OpenJDK8U-jdk_x64_windows_hotspot_8u504b01.msi SHA256
```
It should output a sha256 hash such as:
```text
5115720df210f3c98b592ea2cb9981f48ba6b6942a7c40ba7fe4a59c37d5b815
```
which should match the one on the website.
Now right-click the `.msi` file
Click on `Properties`
Click on the `Digital Signatures` tab
Select the signatures and click on Details
It should say: `The digital signature is OK.`
Install the .msi file.

2\. Checking that Git, Apache Ant and JDK 8 are working:
Open a Command Prompt window (cmd.exe) and run:
```cmd
java -version
```
It should print:
```text
OpenJDK 1.8.0_504
64-Bit Server VM
```
Check if the JDK compiler is available by running:
```cmd
javac -version
```
It should print:
```text
javac 1.8.0_504
```

Press the Windows key on the keyboard
Search for: `environment variables`

Click on `Edit the system environment variables`

In the System Properties window click on `Environment Variables` on the bottom
Under `System Variables`, click on `New`
For `Variable name` type:
```text
ANT_HOME
```
For `Variable value` type:
```text
C:\Tools\apache-ant-1.10.17
```
Now select `Path` in the list

Click on `Edit`

Click on `New`

And type:
```text
%ANT_HOME%\bin
```

Click `OK` on the `Environment Variables` window

Click `OK` on the `System Properties` window

Check whether Ant is configured correctly:
Open a Command Prompt window and run this command:
```cmd
ant -version
```
It should print:
```text
Apache Ant(TM) version 1.10.17 compiled on April 6 2026
```

Open another Command Prompt window and run this command:
```cmd
git --version
```
It should print:
```text
git version 2.55.0.windows.5
```

Now we have these three components working:
- Temurin JDK 8u504
- Apache Ant 1.10.17
- Git 2.55.0

3\. Cloning RSC+

Open a Command Prompt window and run:
```cmd
cd C:\
```
```cmd
mkdir Games
```
```cmd
cd C:\Games
```
Clone the official repository:
```cmd
git clone https://github.com/RSCPlus/rscplus.git
```

When the repository has been cloned, run:
```cmd
cd C:\Games\rscplus
```
and then:
```cmd
dir
```
The file `build.xml` should be listed.

In the Command Prompt window, run:
```cmd
ant dist
```
It should print:
```text
BUILD SUCCESSFUL
```

4\. Checking whether RSC+ works:
Use the following command to run RSC+:
```cmd
java -jar dist\rscplus.jar
```

5\. Downloading the OpenRSC world list:
In a Command Prompt window run this command:
```cmd
cd C:\Games\rscplus
```
Run:
```cmd
java -DdownloadWorlds=openrsc_official -jar dist\rscplus.jar
```

6\. Creating a batch file (and a desktop shortcut for it) in order to run OpenRSC quickly:

In Windows Explorer, navigate to `C:\Games\rscplus`

Right-click on the white space, click on `New`, click on `Text Document`

Rename the `New Text Document.txt` to `Launch RSCPlus.bat`

Right-click on `Launch RSCPlus.bat` and click on `Edit`

In the Notepad window paste these three lines:
```text
@echo off
java -DdownloadWorlds=openrsc_official -jar dist\rscplus.jar
pause
```
Click `File`, click `Save As`
For `Save as type`, select `All Files`
Click on `Save`

Right-click on `Launch RSCPlus.bat`
Click on `Send to` and then click on `Desktop (create shortcut)`
You now have a desktop shortcut for your batch file to start RSCPlus quickly!




