Q= @
E= @echo
# For verbose command line output, uncomment these lines:
#Q=
#E= @:

TEST_SKIPPED="43"

SRCDIR = $(shell find . -type d -not -regex './obj.*' -printf '%P ')
OBJDIR = $(patsubst %,obj/%,$(SRCDIR))

LUASRC = $(shell find . -regex '[^\#]*\.lua' -printf '%P ')
CSRC   = $(shell find . -regex '[^\#]*\.c'   -printf '%P ')
CHDR   = $(shell find . -regex '[^\#]*\.h'   -printf '%P ')
ASM    = $(shell find . -regex '[^\#]*\.dasc' -printf '%P ')
RMSRC  = $(shell find . -name  README.md.src -printf '%P ')

LUAOBJ := $(patsubst %.lua,obj/%_lua.o,$(LUASRC))
COBJ   := $(patsubst %.c,obj/%_c.o,    $(CSRC))
HOBJ   := $(patsubst %.h,obj/%_h.o,    $(CHDR))
ASMOBJ := $(patsubst %.dasc,obj/%_dasc.o,   $(ASM))
JITOBJS:= $(patsubst %,obj/jit_%.o,$(JITSRC))
EXTRAOBJS := obj/jit_tprof.o obj/jit_vmprof.o obj/strict.o
RMOBJS := $(patsubst %.src,%,$(RMSRC))
MDOBJS := $(patsubst %, doc/obj/%.md, $(LUASRC) $(CSRC) $(CHDR)) $(wildcard *.md)

# TESTMODS expands to:
#   core.memory core.lib ...
# for each module that has a top-level selftest () function.
TESTMODS = $(shell find . -regex '[^\#]*\.lua' -printf '%P ' | \
             xargs grep -l '^function selftest *[[:punct:]]' | \
             sed -e 's_\.lua__' -e 's_/_._g')

# TESTSCRIPTS expands to:
#   lib/watchdog/selftest.sh ...
# for each executable selftext.sh script in src.
TESTSCRIPTS = $(shell find . -name "selftest.sh" -executable | xargs)

PATH := ../deps/luajit/usr/local/bin:$(PATH)

all: snabb

markdown: $(RMOBJS)

snabb: $(LUAOBJ) $(HOBJ) $(COBJ) $(ASMOBJ)
	$(E) "LINK      $@"
	$(Q) gcc -Wl,--no-as-needed -Wl,-E -Werror -Wall -o $@ $^ \
	    ../deps/luajit/src/libluajit.a \
	    -lc -ldl -lm -lrt -lpthread
	@echo -n "Firmware: "
	@ln -fs snabb snabbswitch
	@ls -sh snabb

test: $(TESTMODS) $(TESTSCRIPTS)

test_ci: FAIL_ON_FIRST="true"

test_ci: $(TESTMODS) $(TESTSCRIPTS)

$(TESTMODS): testlog snabb
	$(E) "TEST      $@"
	$(Q) ./snabb -t $@ > testlog/$@ 2>&1 || ( \
		EXITCODE="$$?"; \
		[ "$$EXITCODE" -eq $(TEST_SKIPPED) ] \
		&& ( \
			echo "SKIPPED   testlog/$@"; \
			echo "EXITCODE: $$EXITCODE" >> testlog/$@; \
		) \
		|| ( \
			echo "ERROR     testlog/$@"; \
			echo "EXITCODE: $$EXITCODE" >> testlog/$@; \
			if [ -n "$(FAIL_ON_FIRST)" ]; then exit $$EXITCODE; fi;\
		) \
	)

testlog = testlog/$(shell echo "$(@)" | sed -e 's_/_._g')
$(TESTSCRIPTS): testlog snabb
	$(E) "TEST      $@"
	$(Q) ./$@ > $(testlog) 2>&1 || ( \
		EXITCODE="$$?"; \
		[ "$$EXITCODE" -eq $(TEST_SKIPPED) ] \
		&& ( \
			echo "SKIPPED   $(testlog)"; \
			echo "EXITCODE: $$EXITCODE" >> $(testlog); \
		) \
		|| ( \
			echo "ERROR     $(testlog)"; \
			echo "EXITCODE: $$EXITCODE" >> $(testlog); \
			if [ -n "$(FAIL_ON_FIRST)" ]; then exit $$EXITCODE; fi;\
		) \
	)

$(OBJDIR) testlog:
	$(E) "DIR       $@"
	$(Q) mkdir -p $@

$(LUAOBJ): obj/%_lua.o: %.lua Makefile | $(OBJDIR)
	$(E) "LUA       $@"
	$(Q) luajit -bg -n $(subst /,.,$*) $< $@

$(COBJ): obj/%_c.o: %.c $(CHDR) Makefile | $(OBJDIR)
	$(E) "C         $@"
	$(Q) gcc -Wl,-E -I ../deps/luajit/src -I . -c -Wall -Werror -o $@ $<

$(HOBJ): obj/%_h.o: %.h Makefile | $(OBJDIR)
	$(E) "H         $@"
	@(echo -n "module(...,package.seeall); require(\"ffi\").cdef[=============["; \
	 cat $<; \
	 echo "]=============]") > $(basename $@).luah
	$(Q) luajit -bg -n $(subst /,.,$*)_h $(basename $@).luah $@

$(ASMOBJ): obj/%_dasc.o: %.dasc $(CHDR) Makefile | $(OBJDIR)
	$(E) "ASM       $@"
	$(Q) luajit ../deps/luajit/dynasm/dynasm.lua -o $@.gen $<
	$(Q) gcc -Wl,-E -I ../deps/luajit/src -I . -I ../deps/luajit -c -Wall -Werror -x c -o $@ $@.gen

$(JITOBJS): obj/jit_%.o: ../deps/luajit/src/jit/%.lua $(OBJDIR)
	$(E) "LUA       $@"
	$(Q) luajit -bg -n $(patsubst obj/jit_%.o, jit.%, $@) $< $@


$(RMOBJS): %: %.src
	$(E) "MARKDOWN  $@"
	$(Q) scripts/process-markdown $< > $@

# extra/ third party bits and pieces
obj/strict.o: extra/strict.lua | $(OBJDIR)
	$(E) "LUA       $@"
	$(Q) luajit -bg $< $@

obj/jit_tprof.o: extra/tprof.lua | $(OBJDIR)
	$(E) "LUA       $@"
	$(Q) luajit -bg -n jit.tprof $< $@

obj/jit_vmprof.o: extra/vmprof.c | $(OBJDIR)
	$(E) "C         $@"
	$(Q) gcc -Wl,-E -O2 -I ../deps/luajit/src -c -Wall -Werror -o $@ $<


doc/obj/%.lua.md : %.lua Makefile
	awk '/^ *[^-]{3}/ {if (last~/^ *---/) printf("\n")} {print} {last=$$0}' < $< | sed -E -e 's/^/    /g' -e 's/^    --- ?//g' > $@

doc/obj/%.c.md : %.c Makefile
	awk '/^ *[^\/]{3}/ {if (last~/^ *\/\/\//) printf("\n")} {print} {last=$$0}' < $< | sed -E -e 's/^/    /g' -e 's|^    /// ?||g' > $@

doc/obj/%.h.md : %.h Makefile
	awk '/^ *[^\/]{3}/ {if (last~/^ *\/\/\//) printf("\n")} {print} {last=$$0}' < $< | sed -E -e 's/^/    /g' -e 's|^    /// ?||g' > $@

book: doc/snabbswitch.pdf doc/snabbswitch.html doc/snabbswitch.epub

doc/snabbswitch.md: $(MDOBJS) Makefile doc/genbook.sh
	(cd doc; ./genbook.sh) > $@

doc/snabbswitch.pdf: doc/snabbswitch.md
	$(E) "PANDOC    $@"
	$(Q) pandoc --template=doc/template.latex --latex-engine=lualatex -V fontsize=10pt -V monofont=droidsansmono -V monoscale=.70 -V verbatimspacing=.85 -V mainfont=droidserif -V sansfont=droidsans -V documentclass:book -V geometry:top=1.0in -V geometry:bottom=0.75in -S --toc --chapters  -o $@ $<

doc/snabbswitch.html doc/snabbswitch.epub: doc/snabbswitch.md
	$(E) "PANDOC    $@"
	$(Q) pandoc -S --toc --chapters -o $@ $<

CLEAN = snabb snabbswitch obj doc/snabbswitch.* doc/obj/*.md testlog

clean:
	$(E) "RM        $(CLEAN)"
	$(Q)-rm -rf $(CLEAN)

mrproper: clean
	$(E) "RM        $(RMOBJS)"
	$(Q)-rm -rf $(RMOBJS)

.PHONY: clean $(TESTMODS)

