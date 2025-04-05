1. `make menuconfig`: just save and exit
2. `make -j$(nproc)`: if this finishes quickly, it may have failed and output may be higher up
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

5. If compiling an older version of Linux with pahole v1.24 or above, add the following to `scripts/link-vmlinux.sh`:

```
if [ "${pahole_ver}" -ge "124" ]; then
	# see PAHOLE_HAS_LANG_EXCLUDE
	extra_paholeopt="${extra_paholeopt} --skip_encoding_btf_enum64"
fi
```

It should be fairly obvious where to put this if statement, and if it isn't then either this isn't the problem or you'll have to figure out how to pass the `--skip_encoding_btf_enum64` flag to pahole yourself.
