# vfs-jcifs-smb
This project is a vfs provider for the smb protocol.
It can use either:
* [jcifs-codelib](https://github.com/codelibs/jcifs)
* vfs-sandbox

It clones the [vfs-jcifs-smb](https://github.com/new-proimage/vfs-jcifs-smb), that supported jcifs-ng. 
It also contains the vfs-sandbox smb provider.

This Mirrors the approach of jcifs-codelib which combines jcifs-ng and the old smb.

to use the new jcifs-ng implementation you can use the registered schema : "smb".
to use the legacy vfs smb provider and jcifs old implemenation you can use the registered schema : "smb1".

## Why ?
Since the combination of: [jcifs-ng](https://github.com/AgNO3/jcifs-ng) together with [vfs-jcifs-smb](https://github.com/new-proimage/vfs-jcifs-smb)
already provides SMB 1/2/3 implementation why do we need this library.

The reason is that some defaults, or connections behave a bit different between jcifs-ng and the original jcifs when working with SMB1.
I found myself playing with flags like RAW NTLM or disabling the jcifs.smb.client.useSMB2Negotiation, to connect to locations which the original jcifs connected before.

Inorder to provide full backward compatability for those cases, I needed the old jcifs implementation with vfs.

## Maven
```xml
<dependency>
    <groupId>net.new-proimage</groupId>
    <artifactId>vfs-jcifs-smb</artifactId>
    <version>0.9.1</version>
</dependency>
```

## Notes

* You must provide the versions of Commons VFS, and jcifs that you wish to use.
```xml
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-vfs2</artifactId>
    <version>2.7.0</version>
</dependency>
<dependency>
    <groupId>org.codelibs</groupId>
    <artifactId>jcifs</artifactId>
    <version>2.1.19</version>
</dependency>
```
* Commons VFS uses Apache Commons Logging and JCIFS-NG used SLF4J, so to get full logging you need to account for both.
* JCIFS-NG apparently needs Unlimited Crypto enabled for the JVM, but that may depend on servers you are connecting to.
* I didn't implement support for the deprecated practice of putting the credentials in the url. You can provide the
credentials in either the CIFSContext or using StaticUserAuthenticator.
* By default the SingletonContext will be used, but you can provide a customized CIFSContext using
SmbFileSystemConfigBuilder.setCIFSContext()
* An example of using StaticUserAuthenticator for authentication and a custom CIFSContext:
```java
import jcifs.CIFSContext;
import jcifs.CIFSException;
import jcifs.config.PropertyConfiguration;
import jcifs.context.BaseContext;
import org.apache.commons.vfs2.*;
import org.apache.commons.vfs2.auth.StaticUserAuthenticator;
import org.apache.commons.vfs2.impl.DefaultFileSystemConfigBuilder;

import java.net.URI;
import java.net.URISyntaxException;
import java.util.Properties;

public class Example {
    public static void main(String[] args) throws CIFSException, FileSystemException, URISyntaxException {
        if (args.length != 3) {
            System.err.println(" Usage: Example <server> <username> <password>");
            System.exit(-1);
        }

        final String host = args[0];
        final String username = args[1];
        final String password = args[2];

        final URI uri = new URI("smb", host, "/C$", null);

        // authentication
        StaticUserAuthenticator auth = new StaticUserAuthenticator(null, username, password);

        // jcifs configuration
        Properties jcifsProperties = new Properties();

        // these settings are needed for 2.0.x to use anything but SMB1, 2.1.x enables by default and will ignore
        jcifsProperties.setProperty("jcifs.smb.client.enableSMB2", "true");
        jcifsProperties.setProperty("jcifs.smb.client.useSMB2Negotiation", "true");

        CIFSContext jcifsContext = new BaseContext(new PropertyConfiguration(jcifsProperties));

        // pass in both to VFS
        FileSystemOptions options = new FileSystemOptions();
        DefaultFileSystemConfigBuilder.getInstance().setUserAuthenticator(options, auth);
        SmbFileSystemConfigBuilder.getInstance().setCIFSContext(options, jcifsContext);

        final FileSystemManager fsManager = VFS.getManager();
        try (FileObject file = fsManager.resolveFile(uri.toString(), options)) {
            for (FileObject child : file.getChildren()) {
                System.out.println(child.getName());
            }
        }
    }
}
```

