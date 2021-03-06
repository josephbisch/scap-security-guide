include ../../VERSION

IN = input
OUT = output
BUILD = build
TRANS = transforms
SHARED = ../../shared
SHARED_OVAL = $(SHARED)/oval
CONF = ../../config
UTILS = utils
DIST = dist
REFS = $(SHARED)/references

OVAL_5_11 := $(shell oscap --version | grep -q "OVAL Version: 5.11.*"; echo $$?)
OPENSCAP_1_1_OR_LATER := $(shell oscap --version | grep -q -E "OpenSCAP command line tool \(oscap\) 1\.[1-9]+[0-9]*\."; echo $$?)

ID = ssg
PROD = debian8
PROD_OVAL = $(BUILD)/$(PROD)_oval
PROD_REMEDIATIONS = $(BUILD)/$(PROD)_remediations

all: shorthand2xccdf tables guide content dist

shorthand-guide:
	xsltproc -o $(OUT)/$(PROD)-shorthand.xml $(IN)/guide.xslt $(IN)/guide.xml
	xmllint --format --output $(OUT)/$(ID)-$(PROD)-shorthand.xml $(OUT)/$(PROD)-shorthand.xml

shorthand2xccdf: shorthand-guide
	xsltproc --stringparam ssg_version "$(SSG_MAJOR_VERSION).$(SSG_MINOR_VERSION)" -o $(OUT)/unlinked-unresolved-$(PROD)-xccdf.xml $(TRANS)/shorthand2xccdf.xslt $(OUT)/$(PROD)-shorthand.xml
	oscap xccdf resolve -o $(OUT)/unlinked-$(PROD)-empty-groups-xccdf.xml $(OUT)/unlinked-unresolved-$(PROD)-xccdf.xml
	$(SHARED)/$(UTILS)/unselect-empty-xccdf-groups.py --input $(OUT)/unlinked-$(PROD)-empty-groups-xccdf.xml --output $(OUT)/unlinked-$(PROD)-xccdf.xml
	oscap xccdf resolve -o $(OUT)/unlinked-$(PROD)-xccdf.xml $(OUT)/unlinked-$(PROD)-xccdf.xml
	xsltproc -o $(OUT)/unlinked-$(PROD)-ocil.xml $(TRANS)/xccdf-create-ocil.xslt $(OUT)/unlinked-$(PROD)-xccdf.xml
	xmllint --format --output $(OUT)/unlinked-$(PROD)-ocil.xml $(OUT)/unlinked-$(PROD)-ocil.xml
	xsltproc -o $(OUT)/unlinked-ocilrefs-$(PROD)-xccdf.xml $(TRANS)/xccdf-ocilcheck2ref.xslt $(OUT)/unlinked-$(PROD)-xccdf.xml
	# Make intermediate $(PROD_REMEDIATIONS) directory to hold final list of remediation scripts for $(PROD)
	mkdir -p $(PROD_REMEDIATIONS)
	# Search $(SHARED_REMEDIATIONS) and $(IN)/remediations directories to find all product specific remediation scripts,
	# which are regular files (not symlinks). Merge the final list into $(PROD_REMEDIATIONS) directory
	find $(SHARED_REMEDIATIONS) $(IN)/remediations/bash -maxdepth 1 -type f -name *.sh -exec cp {} $(PROD_REMEDIATIONS) ';'
	$(SHARED)/$(TRANS)/combineremediations.py $(CONF) $(PROD) $(PROD_REMEDIATIONS) $(OUT)/bash-remediations.xml
	xsltproc -stringparam remediations "../$(OUT)/bash-remediations.xml" -o $(OUT)/unlinked-$(PROD)-xccdf.xml $(TRANS)/xccdf-addremediations.xslt $(OUT)/unlinked-$(PROD)-xccdf.xml
	xmllint --format --output $(OUT)/unlinked-$(PROD)-xccdf.xml $(OUT)/unlinked-$(PROD)-xccdf.xml

checks:
	# Make intermediate $(PROD_OVAL) directory to hold final list of OVAL checks for $(PROD)
	mkdir -p $(PROD_OVAL)
	# Search $(SHARED_OVAL) and $(IN)/oval directories to find all product specific OVAL checks,
	# which are regular files (not symlinks). Merge the final list into $(PROD_OVAL) directory
	find $(SHARED_OVAL) $(IN)/oval -maxdepth 1 -type f -name *.xml -exec cp {} $(PROD_OVAL) ';'
	# If openscap on the system supports OVAL-5.11 language version, include also OVAL-5.11 checks
	# add local Debian8 specific templated ovals
	$(MAKE) -C $(IN)/oval/templates all
	$(MAKE) -C $(IN)/oval/templates copy
	# into final list of OVAL checks
ifeq ($(OVAL_5_11), 0)
	$(MAKE) -C $(IN)/oval/oval_5.11/templates all
	$(MAKE) -C $(IN)/oval/oval_5.11/templates copy
	find $(IN)/oval/oval_5.11 -maxdepth 1 -type f -name *.xml -exec cp {} $(PROD_OVAL) ';'
	# Update requested debian8 OVAL version in $(CONF)/oval.config to be the 5.11 OVAL language version
	sed -i "s/^\(debian8_oval_version = \).*/\15.11/" $(CONF)/oval\.config
endif
	xmlwf $(PROD_OVAL)/*.xml
	$(SHARED)/$(TRANS)/combineovals.py $(CONF) $(PROD) $(PROD_OVAL) > $(OUT)/unlinked-$(PROD)-oval.xml
	xmllint --format --output $(OUT)/unlinked-$(PROD)-oval.xml $(OUT)/unlinked-$(PROD)-oval.xml

table-refs: shorthand2xccdf
	xsltproc -stringparam ref "nist" -o $(OUT)/table-$(PROD)-nistrefs.html $(TRANS)/xccdf2table-byref.xslt $(OUT)/unlinked-$(PROD)-empty-groups-xccdf.xml
	xsltproc -stringparam profile "common" -o $(OUT)/table-$(PROD)-nistrefs-common.html \
		$(TRANS)/xccdf2table-profilenistrefs.xslt $(OUT)/unlinked-$(PROD)-empty-groups-xccdf.xml
	xsltproc -stringparam profile "ospp-$(PROD)-server" -o $(OUT)/table-$(PROD)-nistrefs-ospp.html \
		$(TRANS)/xccdf2table-profilenistrefs.xslt $(OUT)/unlinked-$(PROD)-empty-groups-xccdf.xml

table-idents: shorthand2xccdf
	xsltproc -o $(OUT)/table-$(PROD)-cces.html $(TRANS)/xccdf2table-cce.xslt $(OUT)/unlinked-$(PROD)-empty-groups-xccdf.xml

table-srgmap: shorthand2xccdf
# the map-to-items filename must be provided relative to the root of the main document being processed
	xsltproc -stringparam map-to-items "../$(OUT)/unlinked-$(PROD)-empty-groups-xccdf.xml" -o $(OUT)/table-$(PROD)-srgmap.html \
		$(TRANS)/table-srgmap.xslt $(REFS)/disa-os-srg-v1r1.xml
	xsltproc -stringparam flat "y" -stringparam map-to-items "../$(OUT)/unlinked-$(PROD)-empty-groups-xccdf.xml" -o $(OUT)/table-$(PROD)-srgmap-flat.html \
		$(TRANS)/table-srgmap.xslt $(REFS)/disa-os-srg-v1r1.xml
	xmllint --xmlout --html --output $(OUT)/table-$(PROD)-srgmap-flat.xhtml $(OUT)/table-$(PROD)-srgmap-flat.html

table-stigs: shorthand2xccdf table-srgmap checks
	xsltproc -o $(OUT)/table-$(PROD)-stig.html $(TRANS)/xccdf2table-stig.xslt $(REFS)/disa-stig-$(PROD)-v1r0.6-xccdf.xml
	xsltproc -o $(OUT)/table-$(PROD)-stig-manual.html $(TRANS)/xccdf2table-stig.xslt $(REFS)/disa-stig-$(PROD)-v1r0.6-xccdf-manual.xml
#	temporarily retain an output file showing the short titles as well
	xsltproc -stringparam profile "stig-$(PROD)-server" -stringparam testinfo "y" -o $(OUT)/table-stig-$(PROD)-testinfo.html \
		$(TRANS)/xccdf2table-profileccirefs.xslt \
		$(OUT)/unlinked-$(PROD)-xccdf.xml
	xsltproc -stringparam overlay "../$(IN)/auxiliary/stig_overlay.xml" -o $(OUT)/unlinked-stig-$(PROD)-xccdf.xml \
		$(TRANS)/xccdf-apply-overlay-stig.xslt \
		$(OUT)/unlinked-$(PROD)-xccdf.xml
	xsltproc -o $(OUT)/table-$(PROD)-stig.html $(TRANS)/xccdf2table-stig.xslt $(OUT)/unlinked-stig-$(PROD)-xccdf.xml

tables: table-refs table-idents table-srgmap

content: shorthand2xccdf checks
#	Remove auxiliary Groups which are only for use in tables, and not guide output.
	xsltproc -o $(OUT)/unlinked-$(PROD)-xccdf-guide.xml $(TRANS)/xccdf-removeaux.xslt $(OUT)/unlinked-$(PROD)-xccdf.xml
#	The relabelids.py script chdirs to ./output, so refer to files from there.
#	Its second argument controls the IDs, as well as the output filenames.
#	Thus, with ID set to ssg, this creates ssg-$(PROD)-xccdf.xml and ssg-$(PROD)-oval.xml.
	$(SHARED)/$(TRANS)/cpe_generate.py $(OUT)/unlinked-$(PROD)-oval.xml $(IN)/oval/platform/$(PROD)-cpe-dictionary.xml $(ID)
	$(SHARED)/$(TRANS)/relabelids.py unlinked-$(PROD)-xccdf.xml $(ID)
	$(SHARED)/$(TRANS)/relabelids.py unlinked-ocilrefs-$(PROD)-xccdf.xml $(ID)
#	Expand 'test_attestation' URLs in OVAL document to valid SSG Contributors wiki link (fixes RHBZ#1155809 for OVAL)
	xsltproc -o $(OUT)/$(ID)-$(PROD)-oval.xml $(TRANS)/oval-fix-test-attestation-urls.xslt $(OUT)/$(ID)-$(PROD)-oval.xml
#	Once things are relabelled, create a datastream
	xsltproc /usr/share/openscap/xsl/xccdf_1.1_remove_dangling_sub.xsl $(OUT)/$(ID)-$(PROD)-xccdf.xml \
		> $(OUT)/$(ID)-$(PROD)-xccdf-nodangles.xml
	xsltproc --stringparam reverse_DNS org.ssgproject.content /usr/share/openscap/xsl/xccdf_1.1_to_1.2.xsl \
		$(OUT)/$(ID)-$(PROD)-xccdf-nodangles.xml > $(OUT)/$(ID)-$(PROD)-xccdf-1.2.xml
	sed -i '/idref="dangling reference to /d' $(OUT)/$(ID)-$(PROD)-xccdf-1.2.xml
	oscap ds sds-compose $(OUT)/$(ID)-$(PROD)-xccdf-1.2.xml $(OUT)/$(ID)-$(PROD)-ds.xml
#	Add in CPE and OVAL content to datastream
	oscap ds sds-add $(OUT)/$(ID)-$(PROD)-cpe-dictionary.xml $(OUT)/$(ID)-$(PROD)-ds.xml
	oscap ds sds-add $(OUT)/$(ID)-$(PROD)-oval.xml $(OUT)/$(ID)-$(PROD)-ds.xml

guide: content
ifeq ($(OPENSCAP_1_1_OR_LATER), 0)
	$(SHARED)/$(UTILS)/build-all-guides.py --input $(OUT)/$(ID)-$(PROD)-ds.xml
else
	@echo "Building guides from XCCDF 1.1, use OpenSCAP 1.1.0 or later for guides from datastreams!"
	$(SHARED)/$(UTILS)/build-all-guides.py --input $(OUT)/$(ID)-$(PROD)-xccdf.xml
endif

validate-xml:
	oscap xccdf validate-xml $(OUT)/$(ID)-$(PROD)-xccdf.xml
	oscap oval validate-xml $(OUT)/$(ID)-$(PROD)-oval.xml
	oscap cpe validate-xml $(OUT)/$(ID)-$(PROD)-cpe-dictionary.xml
#	not yet clean. To be updated
#	oscap oval validate-xml $(OUT)/$(ID)-$(PROD)-cpe-oval.xml
#	oscap ds sds-validate $(OUT)/$(ID)-$(PROD)-ds.xml

validate: validate-xml
	cd $(OUT); ../$(UTILS)/verify-references.py --rules-with-invalid-checks --ovaldefs-unused $(ID)-$(PROD)-xccdf.xml
	oscap oval validate-xml --schematron $(OUT)/$(ID)-$(PROD)-oval.xml

# items in dist are expected for distribution in an rpm
dist: guide content
	mkdir -p $(DIST)/guide $(DIST)/content
	cp $(OUT)/*-guide-*.html $(DIST)/guide
	cp $(OUT)/$(ID)-$(PROD)-xccdf.xml $(DIST)/content
	cp $(OUT)/$(ID)-$(PROD)-oval.xml $(DIST)/content
	cp $(OUT)/$(ID)-$(PROD)-cpe-dictionary.xml $(DIST)/content
	cp $(OUT)/$(ID)-$(PROD)-cpe-oval.xml $(DIST)/content
	cp $(OUT)/$(ID)-$(PROD)-ds.xml $(DIST)/content

eval-common: content
	oscap xccdf eval --profile common $(OUT)/$(ID)-$(PROD)-xccdf.xml

clean:
	rm -f $(OUT)/*.xml $(OUT)/*.html $(OUT)/*.xhtml $(OUT)/*.pdf  $(OUT)/*.spec $(OUT)/*.tar $(OUT)/*.gz $(OUT)/*.ini $(OUT)/*.csv
	rm -rf $(DIST)/content $(DIST)/guide
	$(MAKE) -C $(IN)/oval/templates clean
	$(MAKE) -C $(IN)/oval/oval_5.11/templates clean
	rm -rf $(BUILD)
