---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Verilator Waivers"
subtitle: ""
summary: ""
authors: ["wallento"]
tags: ["Verilator"]
categories: ["Verilator"]
date: 2020-06-12T13:47:41+02:00
lastmod: 2020-06-12T13:47:41+02:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

[Verilator](https://www.veripool.org/wiki/verilator) is a very popular open
source Verilog to C++ compiler and simulator. One functionality of Verilator
that is used increasingly is to use its linter functionality, either locally or
as part of continuous integration. The linter reports problems with your design
that might lead to issues.

With the incredible influx of open source digital designs, I have seen many
projects adding support for Verilator as an open source alternative to
proprietary simulators over the last couple of years. But, not surprisingly,
there are often dozens of warnings emitted by the linter.

Linter warnings could be style issues or actual problems. Hence you would prefer
to use `-Wall` or even `-Wall -Wpedantic` to be sure your code style is just
fine. For example the following Verilog code ([from the Verilator
tests](https://github.com/verilator/verilator/blob/master/test_regress/t/t_lint_blksync_bad.v))
will lead to a linter warning.

```
   always @(posedge clk) begin
      sync_blk = 1'b1;
      sync_blk2 = 1'b1;   // Only warn once per block
      sync_nblk <= 1'b1;
   end
```

```
%Warning-BLKSEQ: t/t_lint_blksync_bad.v:24:16: Blocking assignments (=) in sequential (flop or latch) block
                                             : ... Suggest delayed assignments (<=)
   24 |       sync_blk = 1'b1;
      |                ^
                 ... Use "/* verilator lint_off BLKSEQ */" and lint_on around source to disable this message.
```

The warning is a style warning that can indicate a simulation/synthesis mismatch
due to races in the simulator. In Verilator it will simulate correctly, but it
is generally bad style to do it, hence its a linter warning.

It can be solved in different ways:

1. As suggested by the warning message, the line should be changed to `sync_blk
   <= 1'b1;`. I cannot think of a use case where the original line was intended.
2. You can disable this style warning in general on the command line with
   `-Wno-BLKSEQ`. This has the drawback that it will disable this warning
   globally and you may in future add such a statement accidentally and not
   notice, not nice.
3. As the message suggests as an alternative, you can add a comment around the
   statements to disable the warning for those warnings:

   ```
   always @(posedge clk) begin
      /* verilator lint_off BLKSEQ */
      sync_blk = 1'b1;
      sync_blk2 = 1'b1;
      /* verilator lint_on BLKSEQ */
      sync_nblk <= 1'b1;
   end
   ```

   This is some kind of good solution, in case you and your team have agreed
   that the assignments should not be changed. Probably, you want to add another
   comment why exactly you believe this is a good idea.

   An advantage is that the issue and its solution are directly linked, which is
   a good idea in general. The drawback of this approach is that it needs you
   are touching the code to silence the Verilator linter, especially if you are
   using third-party code. This may be undesirable, especially if your tool zoo
   is large.

## Waiver statements

At last RISC-V workshop I had a breakfast discussion with my fellow [FOSSi
Foundation](https://fossi-foundation.org) director [Philipp
Wagner](https://twitter.com/MrImphil) who works at
[lowRISC](https://lowrisc.org). He suggested there should be better alternatives
to solve this issue. The toy example above obviously doesn't require it, but
there are other warnings and situations when you just want to silence particular
warnings that you have carefully checked and concluded they can be tolerated:
For example unused variable warnings or assignment widths can indicate an issue,
but are often the result of how your design is organized (such as defines and
paramters).

The solution to this is to use a more fine-granular, but non-invasive approach:
*waivers*. A waiver tells a tools that you are aware of a warning, but you are
okay with it. It will thereby suppress warnings that you want to tolerate. It
makes it easier to detect newly introduced errors and you can enter the ideal
world of `-Wall -Wpedantic` while suppressing known warnings.

So, on my way back from San Jose I started the work on waivers in Verilator,
which are part of Verilator since version 4.028. In Verilator you can add
configuration files (typically ending in `.vlt`) that define configuration
settings from a file. What was initially added is the very useful equivalent of
the inline comment from above:

```
`verilator_config

lint_off -rule BLKSEQ -file "t/t_lint_blksync_bad.v" -lines 24-25
```

Just add this file to the build and the warning is waived. This waiver is
already very nice, but there are certain warnings where you want to make sure
that only a particular instance of them is waived. The `-match` flag sets a
known message (with wildcards to keep things simple). If the warning changes,
you will have to revisit it. The first adopter of this syntax is the LowRISC
team for their [ibex](https://github.com/lowRISC/ibex/) core. There is a
[verilator config
file](https://github.com/lowRISC/ibex/blob/master/lint/verilator_waiver.vlt)
that defines the waivers, for example as entire class of warnings (`lint_off
-rule PINCONNECTEMPTY`) or with particular messages that is suppressed
(`lint_off -rule WIDTH -file "*/rtl/ibex_core_tracing.sv" -match "*'RV32M'*"`).

I believe this is a very powerful feature of Verilator and I strongly encourage
projects to use waiver files in combination with `-Wall -Wpedantic`. Even if you
use other simulation or synthesis tools, using Verilator as linter will increase
code quality significantly, in my opinion, which for example is crucial to the
success of open source silicon (but not limited to it, I hope ;).

## Auto-generate waiver file

I have recently looked at many open source designs and found that Verilator is
supported out-of-the-box by nearly all of them, but too often I find `-Wno-lint`
or the rule-based warning suppressions. I wanted to approach the projects and
propose them to use waiver files. But I felt it would be nicer to open a PR
instead that removes the `-Wno-lint` and either solve the issues (oh, boy) or
add an appropriate waiver file with the invitation to review and resolve it. But
for that my bandwidth is just too limited. I felt it was better invested in an
extension to Verilator that automates this process. Hence, since Verilator 4.036
there is a new flag `--waiver-output <filename>` that emits a template for the
waiver file with all warnings emitted.

For example you can find the [regression test](https://github.com/verilator/verilator/blob/master/test_regress/t/t_waiveroutput.v):

```
module t_waiveroutput;

   reg width_warn = 2'b11;
endmodule
```

Running Verilator as linter or normally will give us a width warning and in my
opinion should error (`-Wall -Wpedantic`). Instead of resisting to be pedantic,
lets get the [waiver
file](https://github.com/verilator/verilator/blob/master/test_regress/t/t_waiveroutput.out)
with `--waiver-output`:

```
// DESCRIPTION: Verilator output: Waivers generated with --waiver-output

`verilator_config

// Below you find suggested waivers. You have three options:
//   1. Fix the reason for the linter warning
//   2. Keep the waiver permanently if you are sure this is okay
//   3. Keep the waiver temporarily to suppress the output

// lint_off -rule WIDTH -file "*t/t_waiveroutput.v" -match "Operator ASSIGN expects 1 bits on the Assign RHS, but Assign RHS's CONST '2'h3' generates 2 bits."

// lint_off -rule UNUSED -file "*t/t_waiveroutput.v" -match "Signal is not used: 'width_warn'"
```

The idea is now that you and your team review the individual warnings and either
remove the issue, or un-comment the line and add a comment about why this is a
good idea to waive.

In my opinion this is an excellent starting point to tackle your code quality,
have a solid linter flow. I hope it helps you!

Next, I will start creating PR with waiver files with many TODOs in them :)

## Update: Waiver vs. pragma-style

In the post I have missed to discuss that there is a good reason to use the
inline comments/pragmas versus waiver files. It is certainly a good idea to have
the solution visually next to the problem and this approach is widely used in
teams that use Verilator as one of their main tools. Waivers are especially
useful with third-party code. Overall its also a matter of preference and
waivers are probably preferable when you start using Verilator in a large project.

But, summarized: There is no reason anymore to not use Verilator as Linter in
your project!
