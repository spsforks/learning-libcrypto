## node-mksnapshot Rand issue
This document describes and issue we ran into when dynamically liking Node.js
with quictls/openssl 3.0.0-alpha16. 

The issue occurs as part of the Node.js build and the running of node_mksnapshot
or if Node has already been built running a Node process will appears to "hang".



## Steps to reproduce:
First `OPENSSL_MODULES`, `OPENSSL_CONF`, and `LD_LIBRARY_PATH` need to be
specified as a environment variables, and they can be exported or set for the
command to be run.  This is only required if OpenSSL is installed to a
non-default location.

Here we will export it to make the command to be executed a little shorter.
```console
$ export OPENSSL_MODULES=/home/danielbevenius/work/security/openssl_quic-3.0/lib/ossl-modules/
$ export LD_LIBRARY_PATH=/home/danielbevenius/work/security/openssl_quic-3.0/lib
$ export OPENSSL_CONF=/home/danielbevenius/work/security/openssl_quic-3.0/ssl/openssl.cnf
```
With a correct path specified for the include we can verify that this works
```console
$ ./node --enable-fips -p 'crypto.getFips()'
1
```

Now, if we update openssl.cnf and change the .include path to some non-existing
file the following happens:
```text
.include /bogus/file
```
And then run the same command as above again:
```console
$ ./node --enable-fips -p 'crypto.getFips()'
```
This process will appear to hang and not respond further.

Lets set a breakpoint in CheckEntropy:
```console
$ lldb -- ./out/Debug/node --enable-fips -p 'crypto.getFips()'
(lldb) br s -n CheckEntropy
(lldb) r
(lldb) bt
* thread #1, name = 'node', stop reason = breakpoint 1.1
  * frame #0: 0x000000000130868b node`node::crypto::CheckEntropy() at crypto_util.cc:65:29
    frame #1: 0x00000000013086dd node`node::crypto::EntropySource(buffer="mM\xba\x02", length=8) at crypto_util.cc:78:15
    frame #2: 0x0000000002baa0dc node`v8::base::RandomNumberGenerator::RandomNumberGenerator(this=0x0000000005b44480) at random-number-generator.cc:38:25
    frame #3: 0x0000000002bab7b4 node`v8::base::OS::GetRandomMmapAddr() [inlined] v8::base::LeakyObject<v8::base::RandomNumberGenerator>::LeakyObject<>(this=<unavailable>) at lazy-instance.h:235:5
    frame #4: 0x0000000002bab7aa node`v8::base::OS::GetRandomMmapAddr() at platform-posix.cc:100
    frame #5: 0x0000000002bab7aa node`v8::base::OS::GetRandomMmapAddr() at platform-posix.cc:100
    frame #6: 0x0000000002bab7aa node`v8::base::OS::GetRandomMmapAddr() at platform-posix.cc:274
    frame #7: 0x0000000001703b5a node`v8::internal::Heap::SetUp(this=0x0000000005bbd938) at heap.cc:5142:66
    frame #8: 0x0000000001641772 node`v8::internal::Isolate::Init(this=0x0000000005bb3a60, startup_snapshot_data=0x00007fffffffcbf0, read_only_snapshot_data=0x00007fffffffcc40, can_rehash=<unavailable>) at isolate.cc:3480:14
    frame #9: 0x0000000001d1ca01 node`v8::internal::Snapshot::Initialize(isolate=0x0000000005bb3a60) at snapshot.cc:161:43
    frame #10: 0x00000000013b55b4 node`v8::Isolate::Initialize(isolate=0x0000000005bb3a60, params=0x00007fffffffce40) at api.cc:8420:31
    frame #11: 0x000000000116f2b7 node`node::NodeMainInstance::NodeMainInstance(this=0x00007fffffffcde0, params=0x00007fffffffce40, event_loop=0x0000000005b40e80, platform=0x0000000005bad580, args=size=1, exec_args=size=3, per_isolate_data_indexes=0x0000000005b412f0) at node_main_instance.cc:95:22
    frame #12: 0x00000000010a632a node`node::Start(argc=4, argv=0x00007fffffffd058) at node.cc:1082:43
    frame #13: 0x0000000002812072 node`main(argc=4, argv=0x00007fffffffd058) at node_main.cc:127:21
    frame #14: 0x00007ffff74e81a3 libc.so.6`.annobin_libc_start.c + 243
    frame #15: 0x0000000000fb1d7e node`_start + 46
```
This will lead to Node entering an infinite loop in src/crypto/crypto_util.cc
and the function `EntropySource`:
```c++
void CheckEntropy() {
  for (;;) {
    int status = RAND_status();
    CHECK_GE(status, 0);  // Cannot fail.
    if (status != 0)
      break;
                                                                                
    // Give up, RAND_poll() not supported.
    if (RAND_poll() == 0)
      break;
  }
}
```
Stepping into RAND_status() will land us in crypto/rand/rand_lib.c:
```c
int RAND_status(void)                                                           
{                                                                               
    EVP_RAND_CTX *rand;                                                         
# ifndef OPENSSL_NO_DEPRECATED_3_0                                              
    const RAND_METHOD *meth = RAND_get_rand_method();
    ...
}
```

So when is the OpenSSL configuration file read?  
We can set a break point in conf_def.c (not that quictls/openssl is being used
to the line number need to match that version):
```console
(lldb) br s -f conf_def.c -l 439
(lldb) r
```
This will break in crypto/conf/conf_def.c and the following line:
```c
} else if (strncmp(pname, ".include", 8) == 0                           
                && (p != pname + 8 || *p == '=')) {                                 
                char *include = NULL;                                               
                BIO *next;                                                          
                const char *include_dir = ossl_safe_getenv("OPENSSL_CONF_INCLUDE");
                char *include_path = NULL;                                          
         ...

         next = process_include(include_path, &dirctx, &dirpath);        
                if (include_path != dirpath) {                                  
                    /* dirpath will contain include in case of a directory */   
                    OPENSSL_free(include_path);                                 
                }                               
```
We can verify that this is our specified include file:
```console
(lldb) expr pname
(char *) $0 = 0x0000000005ba7df0 ".include bogus/file"
(lldb) expr include_path
(char *) $7 = 0x0000000005ba9560 "/bogus/file"
```
So `process_include` will be called with the include_path above.
```c
static BIO *process_include(char *include, OPENSSL_DIR_CTX **dirctx,            
                            char **dirpath)                                     
{                                                                               
    struct stat st;                                                             
    BIO *next;                                                                  
                                                                                
    if (stat(include, &st) < 0) {                                               
        ERR_raise_data(ERR_LIB_SYS, errno, "calling stat(%s)", include);        
        /* missing include file is not fatal error */                           
        return NULL;
```
`stat` will fail and an error will be raised on the OpenSSL error stack and
NULL returned. If we step into `ERR_set_error` we can inspect the reason (int)
that is being passed which I think is the value of `errno`
```console
(lldb) expr reason
(int) $0 = 2
```
And we can look that error number up with the following command:
```console
$ errno 2
ENOENT 2 No such file or directory
```

Notice that this is a system error so we cannot use `ERR_reason_error_string`
to see the error reason:
```console
(lldb) br s -n ERR_reason_error_string
(lldb) expr -i false -- ERR_reason_error_string(ERR_peek_error())
```
But we can see the library string:
```
(lldb) call ERR_lib_error_string(ERR_peek_error())
(const char *) $15 = 0x00007ffff7eb6367 "system library"
```
The above information was while being in the `process_include` function, but
when the 
```c
int CONF_modules_load_file_ex(OSSL_LIB_CTX *libctx, const char *filename,          
                              const char *appname, unsigned long flags)         
{                                                                               
    char *file = NULL;                                                          
    CONF *conf = NULL;                                                          
    int ret = 0, diagnostics = 0;                                               
                                                                                
    if (filename == NULL) {                                                     
        file = CONF_get1_default_config_file();                                 
        if (file == NULL)                                                       
            goto err;                                                           
    } else {                                                                    
        file = (char *)filename;                                                
    } 


    ERR_set_mark();
    ...
    conf = NCONF_new_ex(libctx, NULL);                                             
    if (conf == NULL)                                                              
        goto err;                                                                  
                                                                                   
    if (NCONF_load(conf, file, NULL) <= 0) {                                       
        if ((flags & CONF_MFLAGS_IGNORE_MISSING_FILE) &&                           
            (ERR_GET_REASON(ERR_peek_last_error()) == CONF_R_NO_SUCH_FILE)) {   
            ret = 1;                                                               
        }                                                                          
        goto err;                                                                  
    }                  
    
    ret = CONF_modules_load(conf, appname, flags);                                 
    diagnostics = conf_diagnostics(conf);                                          
                                                                                   
  err:                                                                              
    if (filename == NULL)                                                          
        OPENSSL_free(file);                                                        
    NCONF_free(conf);                                                              
                                                                                   
    if ((flags & CONF_MFLAGS_IGNORE_RETURN_CODES) != 0 && !diagnostics)            
        ret = 1;                                                                   
                                                                                   
    if (ret > 0)                                                                   
        ERR_pop_to_mark();                                                         
    else                                                                           
        ERR_clear_last_mark();                                                     
                                                                                   
    return ret;                                                                    
}                  
```
`NCONF` is what is calling `process_include` and notice that there the
error mark is being set before this call. This will then call ERR_pop_to_mark()
which will clear the error.
```
"If CONF_MFLAGS_IGNORE_RETURN_CODES is set the function unconditionally returns
success. This is used by default in OPENSSL_init_crypto(3) to ignore any errors
in the default system-wide configuration file, as having all OpenSSL
applications fail to start when there are potentially minor issues in the file
is too risky. Applications calling CONF_modules_load_file explicitly should not
generally set this flag."
```

We can specify this flag by calling OPENSSL_init_crypto and an example can
be found in [is_fips_enabled.c](../is_fips_enabled.c).

What can be done if check `errno` (there is an example if 
[is_fips_enabled](../is_fips_enabled.c)).


__wip__

