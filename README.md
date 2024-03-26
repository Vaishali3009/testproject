plugins {
    id 'java'
    id 'maven'
    id 'eclipse'
    id 'maven-publish'
}

version = '1.1.0-SNAPSHOT'
group = 'de.telefonica.cct.ccc.shared'
sourceCompatibility = '1.8'

gradle.startParameter.showStacktrace = org.gradle.api.logging.configuration.ShowStacktrace.ALWAYS

sourceSets {
    main {
        java {
            srcDir 'src/'
        }
    }
}

apply from: 'credentials.gradle'
repositories {
    maven{
    	name = 'ccc-releases'
    	url = "https://dot-portal.de.pri.o2.com/nexus/repository/ccc-releases"
		metadataSources {
			mavenPom()
			artifact()
		}
    }
    maven{
    	name = 'tef-public'
    	url = "https://dot-portal.de.pri.o2.com/nexus/repository/public"
		metadataSources {
			mavenPom()
			artifact()
		}
    }
}



dependencies {
    implementation 'com.genesyslab.platform:commons:852.0.3'
    implementation 'com.genesyslab.platform:managementprotocol:852.0.3'
    implementation 'com.genesyslab.platform:protocol:852.0.3'
    compile group: 'log4j', name: 'log4j', version: '1.2.17'
}


def setCredentials() {
    if(project.hasProperty('user') &&  project.getProperty('user') != null){		
        ext.username = project.getProperty('user')
    }else{
        ext.username =  System.getenv('user').toString()
    }
    if(project.hasProperty('password') &&  project.getProperty('password') != null){
        ext.password = project.getProperty('password')
    }else{
        ext.password =  System.getenv('password').toString()
    }
    rootProject.ext.set('User', ext.username)
    rootProject.ext.set('Password', ext.password)
}


test {
    useJUnitPlatform()
    maxHeapSize = '1G'
}

/*publishing {
    publications {
        maven(MavenPublication) {
            artifact("build/libs/LocalControlAgentConnector-$version"+".jar") {
                extension 'jar'
            }}}
    repositories {
        maven {
            name 'tef-snapshots'            
            url "https://dot-portal.de.pri.o2.com/nexus/content/groups/cct-snapshots"
            credentials {
                username nexusUser
                password nexusPassword
            }}}}*/
