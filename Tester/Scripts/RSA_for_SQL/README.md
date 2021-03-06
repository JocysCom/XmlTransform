## RSA Implementation for Microsoft SQL Server and C#
	
Contains C# methods for importing/exporting Microsoft Private Key Format file (.PVK). PVK is a proprietary Microsoft format that stores a cryptographic private key and can be password-protected. PVK files are used by Microsoft SQL Server.

NIST recommended forumla to calculate RSA key strength could be found in "Implementation Guidance for FIPS 140-2 and the Cryptographic Module Validation Program" document, Page 112 - 7.5 Strength of Key Establishment Methods:
https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/fips140-2/fips1402ig.pdf

RSA key strength X-bits:
`calculated_strength_bits=(1.923*POWER(key_length*LN(2),1/3)*POWER(POWER(LN(key_length*LN(2)),2),1/3)-4.69)/LN(2)`

Round strength value:
`rounded_strength_bits=ROUND(calculated_strength_bits/16,0)*16`

RSA key using OAEP padding can encrypt up to X bytes:

`data_in_bytes=(key_length_bits/8) – 42`

`data_out_bytes=(key_length_bits/8)`

Base64 length from output bytes

`data_out_base64_bytes=CEILING((4*data_out_bytes/3)/4,1)*4`

<table>
<tr><th>RSA Key</th><th colspan="2">Strength of Key (bit)</th><th colspan="3">Data (bytes)</th></tr>
<tr><th>Length</th><th>Calculated</th><th>Rounded</th><th>In</th><th>Out</th><th>Out (Base64)</th></tr>
<tr><td align="right">1024</td><td align="right">80.00</td><td align="right">80</td><td align="right">86</td><td align="right">128</td><td align="right">172</td></tr>
<tr><td align="right">2048</td><td align="right">110.12</td><td align="right">112</td><td align="right">214</td><td align="right">256</td><td align="right">344</td></tr>
<tr><td align="right">3072</td><td align="right">131.97</td><td align="right">128</td><td align="right">342</td><td align="right">384</td><td align="right">512</td></tr>
<tr><td align="right">4096</td><td align="right">149.73</td><td align="right">144</td><td align="right">470</td><td align="right">512</td><td align="right">684</td></tr>
<tr><td align="right">7680</td><td align="right">196.25</td><td align="right">192</td><td align="right">918</td><td align="right">960</td><td align="right">1280</td></tr>
<tr><td align="right">15360</td><td align="right">262.62</td><td align="right">256</td><td align="right">1878</td><td align="right">1920</td><td align="right">2560</td></tr>
</table>

## Using faster RSA double encryption to reach 256-bit security

You can generate two key pairs: RSA-3072 (128-bit security) and RSA-4096 (144-bit security), then:

1. Encrypt up to 342-bytes of data with RSA-3072 key using OAEP padding, which will produce 384 bytes.
2. Encrypt data again with RSA-4096 key, which will produce 512 bytes of double encrypted data.

You will get 272-bit (128 + 144) security strength.

Doubling RSA key size increase encryption speed 6-7 times. Therefore, double encryption (3072 and 4096 key) is 20 times faster than encryption with a single 15360 key (256-bit security). Double encryption is perfect for protecting randomly generated AES-256 keys or passwords.

## SQL Script Examples

```TSQL
-- Creating a self-signed certificate.
CREATE CERTIFICATE SqlTestCertificate01   
   ENCRYPTION BY PASSWORD = 'password1234$'  
   WITH SUBJECT = 'SqlTestCertificate01',   
   EXPIRY_DATE = '20201031';  
GO  

-- Backup certificate to files.
BACKUP CERTIFICATE SqlTestCertificate01
	TO FILE = 'c:\ProgramData\MSSQL\SqlTestCertificate01.cer'
	WITH PRIVATE KEY (
		FILE = 'c:\ProgramData\MSSQL\SqlTestCertificate01.pvk',  
		ENCRYPTION BY PASSWORD = 'password1234$',   
		DECRYPTION BY PASSWORD = 'password1234$'
	);

-- Creating a certificate from a file
CREATE CERTIFICATE SqlTestCertificate01   
    FROM FILE = 'c:\ProgramData\MSSQL\SqlTestCertificate01.cer'
    WITH PRIVATE KEY (
		FILE = 'c:\ProgramData\MSSQL\SqlTestCertificate01.pvk',
		DECRYPTION BY PASSWORD = 'password1234$'
	);  
GO 

-- Removing the private key of a certificate.
ALTER CERTIFICATE SqlTestCertificate01 REMOVE PRIVATE KEY  

-- Changing the password that is used to encrypt the private key.
ALTER CERTIFICATE SqlTestCertificate01   
    WITH PRIVATE KEY (DECRYPTION BY PASSWORD = 'password1234$%',  
    ENCRYPTION BY PASSWORD = 'password456$');  
GO

-- Changing the protection of the private key from a password to the database master key.
ALTER CERTIFICATE Shipping15   
    WITH PRIVATE KEY (DECRYPTION BY PASSWORD = '95hk000eEnvjkjy#F%');  
GO 

-- Importing a private key for a certificate that is already present in the database.
ALTER CERTIFICATE SqlTestCertificate01   
    WITH PRIVATE KEY (FILE = 'c:\ProgramData\MSSQL\SqlTestCertificate01.pvk',  
    DECRYPTION BY PASSWORD = 'password1234$');  
GO  

-- Creating a certificate from a file
CREATE CERTIFICATE SqlTestCertificate01   
    FROM FILE = 'c:\ProgramData\MSSQL\SqlTestCertificate01.cer'
    WITH PRIVATE KEY (
		FILE = 'c:\ProgramData\MSSQL\SqlTestCertificate01.pvk',
		DECRYPTION BY PASSWORD = 'password1234$'
	);  
GO 

DECLARE @plain varchar(max) = 'cleartext';
DECLARE @encrypted varbinary(max)
DECLARE @decrypted varchar(max)

SELECT @encrypted = EncryptByCert(Cert_ID('SqlTestCertificate01'), @plain)
SELECT @decrypted = DecryptByCert(Cert_ID('SqlTestCertificate01'), @encrypted, N'password1234$')

SELECT @plain, @decrypted, @encrypted


-- Create symmetric Key and protect with certificate.
CREATE SYMMETRIC KEY SymmetricKey1
WITH ALGORITHM = AES_256
ENCRYPTION BY CERTIFICATE SqlTestCertificate01;

--DROP SYMMETRIC KEY SymmetricKey1

-- Opens the symmetric key for use
OPEN SYMMETRIC KEY SymmetricKey1
DECRYPTION BY CERTIFICATE SqlTestCertificate01
WITH PASSWORD = N'password1234$'

DECLARE @plain2 varchar(max) = 'cleartext';
DECLARE @encrypted2 varbinary(max)
DECLARE @decrypted2 varchar(max)

SELECT @encrypted2 = EncryptByKey(Key_GUID('SymmetricKey1'), @plain2)
SELECT @decrypted2 = DecryptByKey(@encrypted2)

SELECT @plain2, @decrypted2, @encrypted2

-- Closes the symmetric key
CLOSE SYMMETRIC KEY SymmetricKey1
```
