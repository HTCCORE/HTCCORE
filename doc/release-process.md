Release Process
====================

###update (commit) version in sources

	contrib/verifysfbinaries/verify.sh
	doc/README*
	share/setup.nsi
	src/clientversion.h (change CLIENT_VERSION_IS_RELEASE to true)

###tag version in git

	git tag -s v(new version, e.g. 0.8.0)

###write release notes. git shortlog helps a lot, for example:

	git shortlog --no-merges v(current version, e.g. 0.7.2)..v(new version, e.g. 0.10.2.3)

* * *

###update gitian

 In order to take advantage of the new caching features in gitian, be sure to update to a recent version (e9741525c or higher is recommended)

###perform gitian builds

 From a directory containing the Htcoin source, gitian-builder and gitian.sigs.ltc
  
	export SIGNER=(your gitian key, ie wtogami, coblee, etc)
	export VERSION=(new version, e.g. 0.8.0)
	pushd ./Htcoin
	git checkout v${VERSION}
	popd
	pushd ./gitian-builder

###fetch and build inputs: (first time, or when dependency versions change)
 
	mkdir -p inputs

 Register and download the Apple SDK: (see OSX Readme for details)
 
 https://developer.apple.com/downloads/download.action?path=Developer_Tools/xcode_4.6.3/xcode4630916281a.dmg
 
 Using a Mac, create a tarball for the 10.7 SDK and copy it to the inputs directory:
 
	tar -C /Volumes/Xcode/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/ -czf MacOSX10.7.sdk.tar.gz MacOSX10.7.sdk

###Optional: Seed the Gitian sources cache

  By default, gitian will fetch source files as needed. For offline builds, they can be fetched ahead of time:

	make -C ../Htcoin/depends download SOURCES_PATH=`pwd`/cache/common

  Only missing files will be fetched, so this is safe to re-run for each build.

###Build Htcoin Core for Linux, Windows, and OS X:
  
	./bin/gbuild --commit Htcoin=v${VERSION} ../Htcoin/contrib/gitian-descriptors/gitian-linux.yml
	./bin/gsign --signer $SIGNER --release ${VERSION}-linux --destination ../gitian.sigs.ltc/ ../Htcoin/contrib/gitian-descriptors/gitian-linux.yml
	mv build/out/Htcoin-*.tar.gz build/out/src/Htcoin-*.tar.gz ../
	./bin/gbuild --commit Htcoin=v${VERSION} ../Htcoin/contrib/gitian-descriptors/gitian-win.yml
	./bin/gsign --signer $SIGNER --release ${VERSION}-win --destination ../gitian.sigs.ltc/ ../Htcoin/contrib/gitian-descriptors/gitian-win.yml
	mv build/out/Htcoin-*.zip build/out/Htcoin-*.exe ../
	./bin/gbuild --commit Htcoin=v${VERSION} ../Htcoin/contrib/gitian-descriptors/gitian-osx.yml
	./bin/gsign --signer $SIGNER --release ${VERSION}-osx-unsigned --destination ../gitian.sigs.ltc/ ../Htcoin/contrib/gitian-descriptors/gitian-osx.yml
	mv build/out/Htcoin-*-unsigned.tar.gz inputs/Htcoin-osx-unsigned.tar.gz
	mv build/out/Htcoin-*.tar.gz build/out/Htcoin-*.dmg ../
	popd
  Build output expected:

  1. source tarball (Htcoin-${VERSION}.tar.gz)
  2. linux 32-bit and 64-bit binaries dist tarballs (Htcoin-${VERSION}-linux[32|64].tar.gz)
  3. windows 32-bit and 64-bit installers and dist zips (Htcoin-${VERSION}-win[32|64]-setup.exe, Htcoin-${VERSION}-win[32|64].zip)
  4. OSX unsigned installer (Htcoin-${VERSION}-osx-unsigned.dmg)
  5. Gitian signatures (in gitian.sigs/${VERSION}-<linux|win|osx-unsigned>/(your gitian key)/

###Next steps:

Commit your signature to gitian.sigs:

	pushd gitian.sigs
	git add ${VERSION}-linux/${SIGNER}
	git add ${VERSION}-win/${SIGNER}
	git add ${VERSION}-osx-unsigned/${SIGNER}
	git commit -a
	git push  # Assuming you can push to the gitian.sigs tree
	popd

  Wait for OSX detached signature:
	Once the OSX build has 3 matching signatures, Warren/Coblee will sign it with the apple App-Store key.
	He will then upload a detached signature to be combined with the unsigned app to create a signed binary.

  Create the signed OSX binary:

	pushd ./gitian-builder
	# Fetch the signature as instructed by Warren/Coblee
	cp signature.tar.gz inputs/
	./bin/gbuild -i ../Htcoin/contrib/gitian-descriptors/gitian-osx-signer.yml
	./bin/gsign --signer $SIGNER --release ${VERSION}-osx-signed --destination ../gitian.sigs/ ../Htcoin/contrib/gitian-descriptors/gitian-osx-signer.yml
	mv build/out/Htcoin-osx-signed.dmg ../Htcoin-${VERSION}-osx.dmg
	popd

Commit your signature for the signed OSX binary:

	pushd gitian.sigs
	git add ${VERSION}-osx-signed/${SIGNER}
	git commit -a
	git push  # Assuming you can push to the gitian.sigs tree
	popd

-------------------------------------------------------------------------

### After 3 or more people have gitian-built and their results match:

- Perform code-signing.

    - Code-sign Windows -setup.exe (in a Windows virtual machine using signtool)

  Note: only Warren/Coblee has the code-signing keys currently.

- Create `SHA256SUMS.asc` for the builds, and GPG-sign it:
```bash
sha256sum * > SHA256SUMS
gpg --digest-algo sha256 --clearsign SHA256SUMS # outputs SHA256SUMS.asc
rm SHA256SUMS
```
(the digest algorithm is forced to sha256 to avoid confusion of the `Hash:` header that GPG adds with the SHA256 used for the files)

- Upload zips and installers, as well as `SHA256SUMS.asc` from last step, to the Htcoin.ca server

- Update Htcoin.ca version

- Announce the release:

  - Release sticky on Htcointalk: https://Htcointalk.org/index.php?board=1.0

  - Htcoin-development mailing list

  - Update title of #Htcoin on Freenode IRC

  - Optionally reddit /r/Htcoin, ... but this will usually sort out itself

- Add release notes for the new version to the directory `doc/release-notes` in git master

- Celebrate 
