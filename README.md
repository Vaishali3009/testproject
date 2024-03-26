<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>de.telefonica.cct.ccc.shared</groupId>
    <artifactId>your-artifact-id</artifactId>
    <version>1.1.0-SNAPSHOT</version>
    
    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>

    <repositories>
        <repository>
            <id>ccc-releases</id>
            <url>https://dot-portal.de.pri.o2.com/nexus/repository/ccc-releases</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>tef-public</id>
            <url>https://dot-portal.de.pri.o2.com/nexus/repository/public</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>

    <dependencies>
        <dependency>
            <groupId>com.genesyslab.platform</groupId>
            <artifactId>commons</artifactId>
            <version>852.0.3</version>
        </dependency>
        <dependency>
            <groupId>com.genesyslab.platform</groupId>
            <artifactId>managementprotocol</artifactId>
            <version>852.0.3</version>
        </dependency>
        <dependency>
            <groupId>com.genesyslab.platform</groupId>
            <artifactId>protocol</artifactId>
            <version>852.0.3</version>
        </dependency>
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>

</project>
