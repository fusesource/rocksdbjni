# RocksDB JNI

## Description

RocksDB JNI gives you a Java interface to the 
[RocksDB](http://rocksdb.org/) C++ library
which is an embeddable persistent key-value store for fast storage. 

<!-- TODO:
# Getting the JAR

Just add the following jar to your java project:
[rocksdbjni-all-1.0.jar](http://repo2.maven.org/maven2/org/fusesource/rocksdbjni/rocksdbjni-all/1.0/rocksdbjni-all-1.0.jar)

## Using as a Maven Dependency

You just nee to add the following dependency to your Maven pom.

    <dependencies>
      <dependency>
        <groupId>org.fusesource.rocksdbjni</groupId>
        <artifactId>rocksdbjni-all</artifactId>
        <version>1.0</version>
      </dependency>
    </dependencies>
-->

## API Usage:

Recommended Package imports:

    import org.iq80.rocksdb.*;
    import static org.fusesource.rocksdbjni.JniDBFactory.*;
    import java.io.*;

Opening and closing the database.

    Options options = new Options();
    options.createIfMissing(true);
    DB db = factory.open(new File("example"), options);
    try {
      // Use the db in here....
    } finally {
      // Make sure you close the db to shutdown the 
      // database and avoid resource leaks.
      db.close();
    }

Putting, Getting, and Deleting key/values.

    db.put(bytes("Tampa"), bytes("rocks"));
    String value = asString(db.get(bytes("Tampa")));
    db.delete(bytes("Tampa"));

Performing Batch/Bulk/Atomic Updates.

    WriteBatch batch = db.createWriteBatch();
    try {
      batch.delete(bytes("Denver"));
      batch.put(bytes("Tampa"), bytes("green"));
      batch.put(bytes("London"), bytes("red"));

      db.write(batch);
    } finally {
      // Make sure you close the batch to avoid resource leaks.
      batch.close();
    }

Iterating key/values.

    DBIterator iterator = db.iterator();
    try {
      for(iterator.seekToFirst(); iterator.hasNext(); iterator.next()) {
        String key = asString(iterator.peekNext().getKey());
        String value = asString(iterator.peekNext().getValue());
        System.out.println(key+" = "+value);
      }
    } finally {
      // Make sure you close the iterator to avoid resource leaks.
      iterator.close();
    }

Working against a Snapshot view of the Database.

    ReadOptions ro = new ReadOptions();
    ro.snapshot(db.getSnapshot());
    try {
      
      // All read operations will now use the same 
      // consistent view of the data.
      ... = db.iterator(ro);
      ... = db.get(bytes("Tampa"), ro);

    } finally {
      // Make sure you close the snapshot to avoid resource leaks.
      ro.snapshot().close();
    }

Using a custom Comparator.

    DBComparator comparator = new DBComparator(){
        public int compare(byte[] key1, byte[] key2) {
            return new String(key1).compareTo(new String(key2));
        }
        public String name() {
            return "simple";
        }
        public byte[] findShortestSeparator(byte[] start, byte[] limit) {
            return start;
        }
        public byte[] findShortSuccessor(byte[] key) {
            return key;
        }
    };
    Options options = new Options();
    options.comparator(comparator);
    DB db = factory.open(new File("example"), options);
    
Disabling Compression

    Options options = new Options();
    options.compressionType(CompressionType.NONE);
    DB db = factory.open(new File("example"), options);

<!--
Configuring the Cache
    
    Options options = new Options();
    options.cacheSize(100 * 1048576); // 100MB cache
    DB db = factory.open(new File("example"), options);
-->

Getting approximate sizes.

    long[] sizes = db.getApproximateSizes(new Range(bytes("a"), bytes("k")), new Range(bytes("k"), bytes("z")));
    System.out.println("Size: "+sizes[0]+", "+sizes[1]);
    
Getting database status.

    String stats = db.getProperty("rocksdb.stats");
    System.out.println(stats);

<!-- 
Getting informational log messages.

    Logger logger = new Logger() {
      public void log(String message) {
        System.out.println(message);
      }
    };
    Options options = new Options();
    options.logger(logger);
    DB db = factory.open(new File("example"), options);
-->

Destroying a database.
    
    Options options = new Options();
    factory.destroy(new File("example"), options);

Repairing a database.
    
    Options options = new Options();
    factory.repair(new File("example"), options);

Using a memory pool to make native memory allocations more efficient:

    JniDBFactory.pushMemoryPool(1024 * 512);
    try {
        // .. work with the DB in here, 
    } finally {
        JniDBFactory.popMemoryPool();
    }
    
## Building

### Prerequisites 

* GNU compiler toolchain
* [Maven 3](http://maven.apache.org/download.html)

### Supported Platforms

The following worked for me on:

 * OS X Lion with X Code 4
 * CentOS 5.6 (32 and 64 bit)
 * Ubuntu 12.04 (32 and 64 bit)
    * apt-get install autoconf libtool

### Build Procedure

Then download the snappy, rocksdb, and rocksdbjni project source code:

    wget http://snappy.googlecode.com/files/snappy-1.0.5.tar.gz
    tar -zxvf snappy-1.0.5.tar.gz
    git clone git@github.com:facebook/rocksdb.git
    git clone git://github.com/fusesource/rocksdbjni.git
    export SNAPPY_HOME=`cd snappy-1.0.5; pwd`
    export ROCKSDB_HOME=`cd rocksdb; pwd`
    export ROCKSDBJNI_HOME=`cd rocksdbjni; pwd`

<!-- In cygwin that would be
    export SNAPPY_HOME=$(cygpath -w `cd snappy-1.0.5; pwd`)
    export ROCKSDB_HOME=$(cygpath -w `cd rocksdb; pwd`)
    export ROCKSDBJNI_HOME=$(cygpath -w `cd rocksdbjni; pwd`)
-->  

Compile the snappy project.  This produces a static library.    

    cd ${SNAPPY_HOME}
    ./configure --disable-shared --with-pic
    make
    
Patch and Compile the rocksdb project.  This produces a static library.    
    
    cd ${ROCKSDB_HOME}
    export LIBRARY_PATH=${SNAPPY_HOME}
    export C_INCLUDE_PATH=${LIBRARY_PATH}
    export CPLUS_INCLUDE_PATH=${LIBRARY_PATH}
    make librocksdb.a

Now use maven to build the rocksdbjni project.    
    
    cd ${ROCKSDBJNI_HOME}
    mvn clean install

The cd to the platform specific directory that matches your platform

* rocksdbjni-osx
* rocksdbjni-linux32
* rocksdbjni-linux64
* rocksdbjni-win32
* rocksdbjni-win64

And then run:

    mvn clean install

### Build Results

* `rocksdbjni/target/rocksdbjni-${version}.jar` : The java class file to the library.
* `rocksdbjni/target/rocksdbjni-${version}-native-src.zip` : A GNU style source project which you can use to build the native library on other systems.
* `rocksdbjni-${platform}/target/rocksdbjni-${platform}-${version}.jar` : A jar file containing the built native library using your currently platform.
    
