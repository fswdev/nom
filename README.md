# nom, eating data byte by byte

[![LICENSE](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Join the chat at https://gitter.im/Geal/nom](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/Geal/nom?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Build Status](https://github.com/rust-bakery/nom/actions/workflows/ci.yml/badge.svg)](https://github.com/rust-bakery/nom/actions/workflows/ci.yml)
[![Coverage Status](https://coveralls.io/repos/github/rust-bakery/nom/badge.svg?branch=main)](https://coveralls.io/github/rust-bakery/nom?branch=main)
[![crates.io Version](https://img.shields.io/crates/v/nom.svg)](https://crates.io/crates/nom)
[![Minimum rustc version](https://img.shields.io/badge/rustc-1.56.0+-lightgray.svg)](#rust-version-requirements-msrv)


nom是一个用Rust编写的解析器组合库。它的目标是在不影响速度或内存消耗的情况下提供一个构建安全解析器的工具。因此，nom使用了Rust强大的类型和内存安全性来生成快速正确的解析器，并提供函数、宏和trait来抽象大多数容易出错的解析工作。


![nom logo in CC0 license, by Ange Albertini](https://raw.githubusercontent.com/Geal/nom/main/assets/nom.png)

*nom 会很乐意从你的文件中取出一个字节 :)*

<!-- toc -->

- [例子](#示例)
- [文档](#文档)
- [为什么使用 nom?](#why-use-nom)
    - [二进制格式 parsers](#binary-format-parsers)
    - [文本格式 parsers](#text-format-parsers)
    - [编程语言 parsers](#programming-language-parsers)
    - [流格式](#streaming-formats)
- [Parser 组合](#parser-combinators)
- [技术特点](#technical-features)
- [Rust version requirements](#rust-version-requirements-msrv)
- [Installation](#installation)
- [相关的项目](#related-projects)
- [使用nom编写一个 parsers](#parsers-written-with-nom)
- [贡献人](#contributors)

<!-- tocstop -->

## 示例

[Hexadecimal color](https://developer.mozilla.org/en-US/docs/Web/CSS/color) parser:

```rust
use nom::{
  bytes::complete::{tag, take_while_m_n},
  combinator::map_res,
  sequence::Tuple,
  IResult,
  Parser,
};

#[derive(Debug, PartialEq)]
pub struct Color {
  pub red: u8,
  pub green: u8,
  pub blue: u8,
}

fn from_hex(input: &str) -> Result<u8, std::num::ParseIntError> {
  u8::from_str_radix(input, 16)
}

fn is_hex_digit(c: char) -> bool {
  c.is_digit(16)
}

fn hex_primary(input: &str) -> IResult<&str, u8> {
  map_res(
    take_while_m_n(2, 2, is_hex_digit),
    from_hex
  ).parse(input)
}

fn hex_color(input: &str) -> IResult<&str, Color> {
  let (input, _) = tag("#")(input)?;
  let (input, (red, green, blue)) = (hex_primary, hex_primary, hex_primary).parse(input)?;
  Ok((input, Color { red, green, blue }))
}

fn main() {
  println!("{:?}", hex_color("#2F14DF"))
}

#[test]
fn parse_color() {
  assert_eq!(
    hex_color("#2F14DF"),
    Ok((
      "",
      Color {
        red: 47,
        green: 20,
        blue: 223,
      }
    ))
  );
}
```

## 文档

- [Reference documentation](https://docs.rs/nom)
- [The Nominomicon: A Guide To Using Nom](https://tfpk.github.io/nominomicon/)
- [Various design documents and tutorials](https://github.com/fswdev/nom/tree/main/doc)
- [parsers 与 combinators 列表](https://github.com/fswdev/nom/blob/main/doc/choosing_a_combinator.md)

If you need any help developing your parsers, please ping `geal` on IRC (Libera, Geeknode, OFTC), go to `#nom-parsers` on Libera IRC, or on the [Gitter chat room](https://gitter.im/Geal/nom).

## Why use nom

If you want to write:

### Binary format parsers

nom从一开始就被设计用于正确解析二进制格式。与通常的手写C语法分析器相比，
nom语法分析器同样快速，不受缓冲区溢出漏洞，并可以为您处理常见模式:

- [TLV](https://en.wikipedia.org/wiki/Type-length-value)
- Bit level parsing
- Hexadecimal viewer in the debugging macros for easy data analysis
- Streaming parsers for network formats and huge files

Example projects:

- [FLV parser](https://github.com/rust-av/flavors)
- [Matroska parser](https://github.com/rust-av/matroska)
- [tar parser](https://github.com/Keruspe/tar-parser.rs)

### Text format parsers

虽然nom最初是为二进制格式而设计的，但很快nom就适用于文本格式的解析。
从基于行的简单格式(CSV等)，到复杂的嵌套格式(JSON), nom都可以处理它，并为您提供有用的工具：

- Fast case insensitive comparison
- Recognizers for escaped strings
- Regular expressions can be embedded in nom parsers to represent complex character patterns succinctly
- Special care has been given to managing non ASCII characters properly

Example projects:

- [HTTP proxy](https://github.com/sozu-proxy/sozu/tree/main/lib/src/protocol/http/parser)
- [TOML parser](https://github.com/joelself/tomllib)

### Programming language parsers

虽然编程语言解析器通常是手动编写的，以获得更大的灵活性和性能，但nom可以（并且已经成功地）用作语言的原型解析器。

nom将让您快速开始使用强大的自定义错误类型，您可以使用[nom_locate](https://github.com/fflorent/nom_locate)
以精确定位错误的确切行和列。不需要单独的标记化、词法分析和解析阶段：nom可以自动处理空白解析，并就地构建AST。

Example projects:

- [PHP VM](https://github.com/tagua-vm/parser)
- eve language prototype
- [xshade shading language](https://github.com/xshade-lang/xshade)

### Streaming formats

虽然很多格式（以及处理它们的代码）都认为它们可以容纳内存中的完整数据，但也有一些格式我们一次只能获得部分数据，
比如网络格式或大文件。
nom是为部分数据的正确行为而设计的：如果没有足够的数据来决定，nom会告诉你它需要更多的数据，
而不是默默地返回错误的结果。无论你的数据是完整的还是大块的，结果都应该是一样的。

它允许您为您的协议构建强大的、确定性的状态机。

Example projects:

- [HTTP proxy](https://github.com/sozu-proxy/sozu/tree/main/lib/src/protocol/http/parser)
- [Using nom with generators](https://github.com/rust-bakery/generator_nom)

## Parser combinators

语法分析器组合子是一种语法分析器的方法，
与[lex](https://en.wikipedia.org/wiki/Lex_(software)) 和
[yacc](https://en.wikipedia.org/wiki/Yacc)等软件完全不同。
您不需要在单独的文件中编写语法来生成相应的代码，只需要使用具有特定目的的非常小的函数，如“占用5个字节”，
或者“识别‘HTTP’这个词”，并将它们组合成有意义的模式，比如“识别‘HTTP'，然后是一个空格，然后是版本”。
生成的代码很小，效果就像用其他解析器方法编写的语法一样。
 
nom的几个优点：
- 解析器很小，编写起来很容易
- 解析器组件易于重用（如果它们足够通用，请将它们添加到nom！）
- 解析器组件易于单独测试（单元测试和基于属性的测试）
- 解析器组合代码看起来与您编写的语法非常接近
- 您可以针对当前需要的数据构建部分解析器，并忽略其余部分

## Technical features

nom解析器用于：

- [x]**byte-oriented**：基本类型为“&[u8]”，解析器将尽可能多地在字节数组切片上工作（但不限于此）
- [x]**bit-oriented**：nom可以将字节片寻址为位流
- [x]**string-oriented**：同样类型的组合子也可以应用于UTF-8字符串
- [x]**zero-copy**：如果解析器返回其输入数据的子集，它将返回该输入的切片，而不进行复制
- [x]**streaming**:nom可以处理部分数据，并检测何时需要更多数据才能产生正确的结果
- [x]**descriptive errors**：解析器可以聚合一个错误代码列表，其中包含指向递增输入切片的指针。这些错误列表可以进行模式匹配，以提供有用的消息。
- [x]**custom error types**：您可以提供特定的类型来改进解析器返回的错误
- [x]**safe parsing**:nom利用了Rust的安全内存处理和强大的类型，解析器通常会被模糊化，并用真实世界的数据进行测试。到目前为止，模糊化发现的唯一缺陷是在nom之外编写的代码中
- [x]**speed**：基准测试表明，nom解析器通常优于许多解析器组合子库，如Parsec和attoparsec，一些正则表达式引擎，甚至手写C解析器
 

Some benchmarks are available on [GitHub](https://github.com/rust-bakery/parser_benchmarks).

## Rust version requirements (MSRV)

The 7.0 series of nom supports **Rustc version 1.56 or greater**.

The current policy is that this will only be updated in the next major nom release.

## Installation

nom is available on [crates.io](https://crates.io/crates/nom) and can be included in your Cargo enabled project like this:

```toml
[dependencies]
nom = "7"
```

There are a few compilation features:

* `alloc`: (activated by default) if disabled, nom can work in `no_std` builds without memory allocators. If enabled, combinators that allocate (like `many0`) will be available
* `std`: (activated by default, activates `alloc` too) if disabled, nom can work in `no_std` builds

You can configure those features like this:

```toml
[dependencies.nom]
version = "7"
default-features = false
features = ["alloc"]
```

# Related projects

- [Get line and column info in nom's input type](https://github.com/fflorent/nom_locate)
- [Using nom as lexer and parser](https://github.com/Rydgel/monkey-rust)

# Parsers written with nom

Here is a (non exhaustive) list of known projects using nom:

- Text file formats:
[Ceph Crush](https://github.com/cholcombe973/crushtool),
[Cronenberg](https://github.com/ayrat555/cronenberg),
[XFS Runtime Stats](https://github.com/ChrisMacNaughton/xfs-rs),
[CSV](https://github.com/GuillaumeGomez/csv-parser),
[FASTA](https://github.com/TianyiShi2001/nom-fasta),
[FASTQ](https://github.com/elij/fastq.rs),
[INI](https://github.com/rust-bakery/nom/blob/main/tests/ini.rs),
[ISO 8601 dates](https://github.com/badboy/iso8601),
[libconfig-like configuration file format](https://github.com/filipegoncalves/rust-config),
[Web archive](https://github.com/sbeckeriv/warc_nom_parser),
[PDB](https://github.com/TianyiShi2001/nom-pdb),
[proto files](https://github.com/tafia/protobuf-parser),
[Fountain screenplay markup](https://github.com/adamchalmers/fountain-rs),
[vimwiki](https://github.com/chipsenkbeil/vimwiki-rs/tree/master/vimwiki) & [vimwiki_macros](https://github.com/chipsenkbeil/vimwiki-rs/tree/master/vimwiki_macros)
- Programming languages:
[PHP](https://github.com/tagua-vm/parser),
[Basic Calculator](https://github.com/balajisivaraman/basic_calculator_rs),
[GLSL](https://github.com/phaazon/glsl),
[Lua](https://github.com/rozbb/nom-lua53),
[Python](https://github.com/ProgVal/rust-python-parser),
[SQL](https://github.com/ms705/nom-sql),
[Elm](https://github.com/cout970/Elm-interpreter),
[SystemVerilog](https://github.com/dalance/sv-parser),
[Turtle](https://github.com/vandenoever/rome/tree/master/src/io/turtle),
[CSML](https://github.com/CSML-by-Clevy/csml-engine/tree/dev/csml_interpreter),
[Wasm](https://github.com/fabrizio-m/wasm-nom),
[Pseudocode](https://github.com/Gungy2/pseudocod),
[Filter for MeiliSearch](https://github.com/meilisearch/meilisearch)
- Interface definition formats: [Thrift](https://github.com/thehydroimpulse/thrust)
- Audio, video and image formats:
[GIF](https://github.com/Geal/gif.rs),
[MagicaVoxel .vox](https://github.com/dust-engine/dot_vox),
[MIDI](https://github.com/derekdreery/nom-midi-rs),
[SWF](https://github.com/open-flash/swf-parser),
[WAVE](https://github.com/Noise-Labs/wave),
[Matroska (MKV)](https://github.com/rust-av/matroska)
- Document formats:
[TAR](https://github.com/Keruspe/tar-parser.rs),
[GZ](https://github.com/nharward/nom-gzip),
[GDSII](https://github.com/erihsu/gds2-io)
- Cryptographic formats:
[X.509](https://github.com/rusticata/x509-parser)
- Network protocol formats:
[Bencode](https://github.com/jbaum98/bencode.rs),
[D-Bus](https://github.com/toshokan/misato),
[DHCP](https://github.com/rusticata/dhcp-parser),
[HTTP](https://github.com/sozu-proxy/sozu/tree/main/lib/src/protocol/http),
[URI](https://github.com/santifa/rrp/blob/master/src/uri.rs),
[IMAP](https://github.com/djc/tokio-imap),
[IRC](https://github.com/Detegr/RBot-parser),
[Pcap-NG](https://github.com/richo/pcapng-rs),
[Pcap](https://github.com/ithinuel/pcap-rs),
[Pcap + PcapNG](https://github.com/rusticata/pcap-parser),
[IKEv2](https://github.com/rusticata/ipsec-parser),
[NTP](https://github.com/rusticata/ntp-parser),
[SNMP](https://github.com/rusticata/snmp-parser),
[Kerberos v5](https://github.com/rusticata/kerberos-parser),
[DER](https://github.com/rusticata/der-parser),
[TLS](https://github.com/rusticata/tls-parser),
[IPFIX / Netflow v10](https://github.com/dominotree/rs-ipfix),
[GTP](https://github.com/fuerstenau/gorrosion-gtp),
[SIP](https://github.com/kurotych/sipcore/tree/master/crates/sipmsg),
[Prometheus](https://github.com/vectordotdev/vector/blob/master/lib/prometheus-parser/src/line.rs)
- Language specifications:
[BNF](https://github.com/shnewto/bnf)
- Misc formats:
[Game Boy ROM](https://github.com/MarkMcCaskey/gameboy-rom-parser),
[ANT FIT](https://github.com/stadelmanma/fitparse-rs),
[Version Numbers](https://github.com/fosskers/rs-versions),
[Telcordia/Bellcore SR-4731 SOR OTDR files](https://github.com/JamesHarrison/otdrs),
[MySQL binary log](https://github.com/PrivateRookie/boxercrab),
[URI](https://github.com/Skasselbard/nom-uri),
[Furigana](https://github.com/sachaarbonel/furigana.rs),
[Wordle Result](https://github.com/Fyko/wordle-stats/tree/main/parser)

Want to create a new parser using `nom`? A list of not yet implemented formats is available [here](https://github.com/rust-bakery/nom/issues/14).

Want to add your parser here? Create a pull request for it!

# Contributors

nom is the fruit of the work of many contributors over the years, many thanks for your help!

<a href="https://github.com/rust-bakery/nom/graphs/contributors">
  <img src="https://contributors-img.web.app/image?repo=rust-bakery/nom" />
</a>
