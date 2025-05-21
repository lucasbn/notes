1. `make menuconfig`: just save and exit
2. `make -j$(nproc)`: if this finishes quickly, it may have failed and output may be higher up
	- `KBUILD_BUILD_TIMESTAMP='' make "CC=ccache gcc" -j$(nproc)` massively speeds up build
		times with caching
3. `No rule to make target 'debian/canonical-certs.pem` can be fixed by running:
	- `scripts/config --disable SYSTEM_TRUSTED_KEYS`
	- `scripts/config --disable SYSTEM_REVOCATION_KEYS`
4. GCC 12 and above detects a free after use error in `subcmd-util.h` for older kernel versions, fix with:

```
static inline void *xrealloc(void *ptr, size_t size)
{
	void *ret = realloc(ptr, size);
	if (!ret)
		die("Out of memory, realloc failed");
	return ret;
}
```

5. If you see any error like `FAILED: load BTF from vmlinx` and you are compiling an older version of the kernel with pahole v1.24 or above, you'll need to add a flag to the invocation of pahole: 

You may be able to do this in `scripts/link-vmlinux.sh` (depending on the kernel version):

```
if [ "${pahole_ver}" -ge "124" ]; then
	# see PAHOLE_HAS_LANG_EXCLUDE
	extra_paholeopt="${extra_paholeopt} --skip_encoding_btf_enum64"
fi
```

Alternatively, you can add the flag like this:

```
LLVM_OBJCOPY=${OBJCOPY} ${PAHOLE} -J --skip_encoding_btf_enum64 ${1}
```

Hopefully it'll be clear where to add this flag (`--skip_encoding_btf_enum64`), but you may need to do some digging/experimenting 

6. To install pahole (if you get a `BTF: .tmp_vmlinux.btf: pahole (pahole) is not available` error) run `sudo apt install dwarves`

7. To install modules:

```
sudo make modules_install INSTALL_MOD_PATH=modules
sudo mount image/rootfs.ext4 /mnt/tmpmnt
sudo cp -a modules/lib/modules /mnt/tmpmnt/lib/

```

