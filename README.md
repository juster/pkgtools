# ArchLinux Packager Tools

These scripts help me deal with the drudgery of maintaining packages for
ArchLinux's official repositories. Some of them are general purpose so you
might find other uses for them. Others are more specifically suited to the
task of maintaining perl packages.

In my attempt at consistancy, return values and behavior adhere to
the following self-imposed rules:

* Successful program execution exits with code 0 (duh).
* Messages intended for humans are printed to stderr.
* If no program arguments are supplied, a *very* minimal usage message is
  printed to stderr and the program exits with code 2.

# External Requirements

Curl is required for all interaction with FTP or web sites. The other requirements
are optional if you don't use the scripts requiring them.

GraphViz graph rendering programs are required to generate the graph images. My
own [genpkg](https://github.com/juster/genpkg) package generation program
is used to convert dependencies from external sources (i.e. CPAN) to our internal
package representation. This is only used by tododeps.

# Installation

Install these scripts by either copying them to a directory in your path (i.e. ~/bin)
or by adding the directories containing them to your runtime PATH.

# Example Usage

	Find out of date perl packages that juster maintains:
	webpkgs -m juster | perlood | tododeps juster | tee deps.todo
	
	Draw a graph of all dependencies we need to build:
	cat deps.todo | resdeps | depdot | circo -Tpng > deps.png

# General Purpose Tools

The target of these tools are binary packages released by ArchLinux. They
deal with the outermost layer of abstraction that is revealed to the public.
While findpkgs deals with the raw files themselves, the webpkgs tool deals
with the website/database of package release information. This website
contains data that is not encoded into the package binary and can be
considered more complete. The data available in the package files
themselves is a subset of the data available on the website.

## fetchpkgs

Usage:

	fetchpkgs [package names]

Description:

Give this script package names as arguments and it will download them from
the ftp.archlinux.org mirror. Prints the repo, package name, and filename
for each argument found, not necessarily in the same order as the lines
of input were received.

Environment:

* ARCH (defaults to "i686")

Returns:

* 101 if FTP listing fails
* 102 if FTP download fails
* 103 if package arguments were not found, which are printed to stderr

Requires:

* curl

## findpkgs

Usage:

	findpkgs [package names]

Description:

This script came after fetchpkgs and only prints the filename of found
packages on the FTP repos. Prints the package name, repo, and filename
for each package found. Package names can be given in an absololute form,
with the repo and package name seperated by a forward-slash (i.e. core/perl).
If this occurs that particular package is *only* searched for in the specified repo.

I should rewrite fetchpkgs to use this script.

Environment:

* MIRROR (defaults to "mirrors.kernel.org/archlinux")
* ARCH (defaults to "i686")
* REPOS (defaults to "core extra community")

Returns:

* 101 if FTP listing fails
* 102 if a package not specified in absolute form was not found. These names are printed to stderr.
* 103 if a package specified in absolute form was not found. For each repo specified explicitly, the
  names of packages that were not found in that repo are printed on a line to stderr.

Requires:

* curl

## webpkgs

Usage:

	webpkgs [-m person] [-r repo] [search string]

Description:

Queries the JSON interface at <https://packages.archlinux.org>. Keep in mind the
website limits the number of results to 250 results as of this writing. Specify a
maintainer, repo, and/or search string. Prints the package name, version,
repo, arch, and maintainers. Multiple archs are printed more than once. If
the package is flagged out of date an asterisk is appended to the version.
There can be one or many maintainers. No maintainers means the package
is orphaned.

Returns:

* 101 if curl exits with an error.
* 102 if no results were found (JSON says it is an invalid match).
* 103 if unsafe chars are used in the query.

Requires:

* curl

# Perl Package Scripts

I poorly maintain perl packages so here are scripts for dealing with that
grand role. These are placed inside the perl subdirectory.

## cpandists

Usage:

	cpandists > cpan.dist.list

Description:
Fetches a list of CPAN distributions from <http://cpan.pair.com>. Prints the
distribution name and its version for every distribution on CPAN.

Returns:

* 101 if curl or gzip failed and there were no distributions printed.

Requires:

* curl

## perlood

Usage:

	webpkgs -m juster | perlood | tee pkgs.ood

Description:

Reads a list of packages, their versions, and etc from standard input.
Pipes in a list of cpan distributions from the cpandists script. This will
guess which packages correspond to which CPAN distributions and print out
the lines for packages it thinks are out of date. This performs no complex
version comparison but simply checks for string or numeric mismatch.

Requires:

* cpandists

## tododeps

Usage:

	webpkgs -m juster | perlood | tododeps juster | tee pkgs.todo

Description:

My real goal here. Reads package names, versions, and repos from standard
input. Writes a list of each package and that package's dependencies. The
dependencies are traversed until the recursion is exhausted. Packages owned
by people other than the supplied maintainer are not printed or traversed.
Only orphaned or missing dependencies are traversed. All packages owned by
the maintainer are assumed to be fed as input.

This runs genpkg to find dependencies of a package. This means that only deps
that genpkg knows how to generate (i.e. perl module packages) will succeed in
this process. Calling genpkg generates source packages at the same time.

Requires:

* [genpkg](https://github.com/juster/genpkg)
* webpkgs

## resdeps

Usage:

	webpkgs -m juster | perlood | tododeps juster | resdeps | tee pkgdeps.todo

Description:

Resolves dependencies. Standard input is a dependency list, like tododeps's
output. Resolves all package names into their [repo]/[package] absolute
name. This makes it easier to graph them.

# Dependency Graphing

Data visualization makes this torrid mess easier. I use GraphViz to plot the
graph images.

## depdot

Usage:

	webpkgs -m juster | perlood | tododeps juster | resdeps | depdot | circo -Tpng > deps.png

Feed this the output of resdeps to create a .dot file that can be fed into
a GraphViz graphing program.

# Author

Justin "juster" Davis
<jrcd83@gmail.com>
