# Entry point for Copr SCM builds with --method make_srpm.
#
# Copr invokes exactly:  make -f .copr/Makefile srpm outdir=<dir> spec=<path>
# The srpm-build chroot only provides curl + rpm-build (no rpmdevtools/spectool),
# so the remote Source0 URL is expanded with rpmspec and fetched with curl here.
# This keeps obsidian.spec the single source of truth: bumping Version: is enough.

spec ?= obsidian.spec
outdir ?= $(CURDIR)

.PHONY: srpm
srpm:
	@set -euo pipefail; \
	url=$$(rpmspec -P "$(spec)" | sed -n 's/^Source0:[[:space:]]*//p' | head -1); \
	if [ -z "$$url" ]; then echo "error: no Source0 URL found in $(spec)" >&2; exit 1; fi; \
	echo "Fetching $$url"; \
	curl -fL --retry 3 --retry-delay 5 -o "$(CURDIR)/$${url##*/}" "$$url"; \
	rpmbuild -bs \
	    --define "_sourcedir $(CURDIR)" \
	    --define "_specdir $(CURDIR)" \
	    --define "_builddir $(CURDIR)/.build" \
	    --define "_buildrootdir $(CURDIR)/.buildroot" \
	    --define "_srcrpmdir $(outdir)" \
	    "$(spec)"
