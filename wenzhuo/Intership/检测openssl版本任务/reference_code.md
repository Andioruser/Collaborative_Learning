boyang.lai mentor写了一个参考版本的代码，供我学习。 

原话：你的代码不错，可以用，但是不易用，可以看一下我写的代码，学习一下。

```rust
// probe.rs
use std::borrow::Cow;
// config.rs
use std::fs::{self, File};
use std::io::{self, BufRead, BufReader, Read};
use std::path::{Path, PathBuf};
use std::{fmt, str};

use elf::endian::AnyEndian;
use elf::{ElfStream, ParseError};
use glob::glob;
use regex::bytes::Regex;
use tracing::{debug, info, warn};

const LD_LOAD_PATH: &str = "/etc/ld.so.conf";
const DEFAULT_SO_PATHS: [&str; 4] = ["/lib", "/usr/lib", "/usr/lib64", "/lib64"];
const OPENSSL_VERSION_REGEX: &str = r"(OpenSSL\s\d\.\d\.[0-9a-z]+)";
const OPENSSL_VERSION_LEN: usize = 30; // This value can be adjusted based on the actual buffer length
//const DEFAULT_IFNAME: &str = "eth0";
const LIBSSL_SHARED_OBJECTS: [&str; 3] = ["libssl.so.3", "libssl.so.1.1", "libssl.so.10"];

pub const PROBE_OPENSSL_FN: [&str; 4] = [
    "SSL_get_wbio",
    "SSL_in_before",
    "SSL_do_handshake",
    "SSL_verify_client_post_handshake",
];

#[derive(Debug)]
pub enum OpensslProbeError {
    Io(io::Error),
    ElfParseError(ParseError),
    UnsupportedArchitecture(String),
    SectionNotFound,
    VersionNotFound,
    RegexError(regex::Error),
}

impl std::error::Error for OpensslProbeError {}

impl From<io::Error> for OpensslProbeError {
    fn from(err: io::Error) -> Self {
        OpensslProbeError::Io(err)
    }
}

impl From<ParseError> for OpensslProbeError {
    fn from(err: ParseError) -> Self {
        OpensslProbeError::ElfParseError(err)
    }
}

impl From<regex::Error> for OpensslProbeError {
    fn from(err: regex::Error) -> Self {
        OpensslProbeError::RegexError(err)
    }
}

impl fmt::Display for OpensslProbeError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            OpensslProbeError::Io(e) => write!(f, "IO error: {}", e),
            OpensslProbeError::ElfParseError(e) => write!(f, "ELF parsing error: {}", e),
            OpensslProbeError::UnsupportedArchitecture(a) => {
                write!(f, "Unsupported architecture: {}", a)
            },
            OpensslProbeError::SectionNotFound => write!(f, "Section not found"),
            OpensslProbeError::VersionNotFound => write!(f, "Version not found"),
            OpensslProbeError::RegexError(e) => write!(f, "Regex error: {}", e),
        }
    }
}

// Check OpenSSL configuration, looking for `libssl.so`
pub fn check_openssl() -> Result<PathBuf, OpensslProbeError> {
    let so_load_paths = get_dyn_lib_dirs();
    debug!(so_load_paths = ?so_load_paths, "Searching for OpenSSL library");
    for so_path_entry in so_load_paths {
        for &so_file in &LIBSSL_SHARED_OBJECTS {
            let ssl_path = Path::new(&so_path_entry).join(so_file);
            if ssl_path.exists() {
                return Ok(ssl_path);
            }
        }
    }
    Err(OpensslProbeError::Io(io::Error::new(
        io::ErrorKind::NotFound,
        "Unable to find OpenSSL library",
    )))
}

/// Reads and parses dynamic library configuration files based on the given pattern
/// and returns a list of directories found in them (or an error).
pub fn parse_dyn_lib_conf<P: AsRef<Path>>(
    origin: P,
    pattern: &str,
) -> Result<Vec<String>, OpensslProbeError> {
    let mut dirs = Vec::new();
    let pattern = if !pattern.starts_with('/') {
        Cow::Owned(format!(
            "{}/{}",
            origin.as_ref().parent().unwrap().to_str().unwrap(),
            pattern
        ))
    } else {
        Cow::Borrowed(pattern)
    };
    let files = glob(&pattern).map_err(|e| {
        OpensslProbeError::Io(io::Error::new(
            io::ErrorKind::Other,
            format!("Error collecting files: {}", e),
        ))
    })?;

    for file in files {
        let config_file = file.map_err(|e| {
            OpensslProbeError::Io(io::Error::new(
                io::ErrorKind::Other,
                format!("Error collecting files: {}", e),
            ))
        })?;
        debug!(config_file = ?config_file.display(), pattern = pattern.as_ref(), "Found ld.so conf files");
        let configfile = config_file.to_string_lossy();
        if configfile.contains("lib32") {
            continue;
        }

        let file = fs::File::open(&config_file).map_err(OpensslProbeError::Io)?;
        let reader = BufReader::new(file);

        for line in reader.lines() {
            let line = line.map_err(OpensslProbeError::Io)?;
            let line = line.trim();

            // Ignore comments and empty lines
            if line.is_empty() || line.starts_with('#') || line.starts_with(';') {
                continue;
            }

            // Found "include" directive
            let words: Vec<&str> = line.split_whitespace().collect();
            if words
                .first()
                .is_some_and(|&word| word.to_lowercase() == "include")
            {
                if let Ok(subdirs) = parse_dyn_lib_conf(&config_file, words[1].trim()) {
                    dirs.extend(subdirs);
                }
            } else {
                dirs.push(line.to_string());
            }
        }
    }

    if dirs.is_empty() {
        Err(OpensslProbeError::Io(io::Error::new(
            io::ErrorKind::Other,
            format!("read keylogger :{} error.", pattern),
        )))
    } else {
        Ok(dirs)
    }
}

/// Returns the list of directories to search for dynamic libraries,
/// either from the configuration or a default list.
pub fn get_dyn_lib_dirs() -> Vec<String> {
    match parse_dyn_lib_conf(LD_LOAD_PATH, LD_LOAD_PATH) {
        Ok(mut dirs) => {
            dirs.push("/lib64".to_string());
            dirs.push("/usr/lib64".to_string());
            dirs
        },
        Err(_) => DEFAULT_SO_PATHS
            .iter()
            .map(|&path| path.to_string())
            .collect(),
    }
}
pub fn detect_openssl_version_by_path<P: AsRef<Path>>(
    so_path: P,
) -> Result<String, OpensslProbeError> {
    // Open file
    let mut f = File::open(so_path).map_err(OpensslProbeError::Io)?;

    // Read file contents into byte array
    let buffer = {
        let mut buffer = Vec::new();
        f.read_to_end(&mut buffer).map_err(OpensslProbeError::Io)?;
        buffer
    };

    // Parsing ELF files
    let mut elf_file: ElfStream<AnyEndian, &mut File> =
        ElfStream::open_stream(&mut f).map_err(OpensslProbeError::ElfParseError)?;
    // Check machine
    match elf_file.ehdr.e_machine {
        elf::abi::EM_X86_64 | elf::abi::EM_AARCH64 => {},
        _ => {
            return Err(OpensslProbeError::UnsupportedArchitecture(format!(
                "Unsupported arch library, ELF Header Machine is: {:?}",
                elf_file.ehdr.e_machine
            )));
        },
    }

    // Find the .rodata section
    let rodata_section = elf_file
        .section_header_by_name(".rodata")
        .map_err(OpensslProbeError::ElfParseError)?
        .ok_or(OpensslProbeError::SectionNotFound)?;

    let section_offset = rodata_section.sh_offset as usize;
    let section_size = rodata_section.sh_size as usize;

    // Find OpenSSL version using regular expression
    let rex = Regex::new(OPENSSL_VERSION_REGEX).map_err(OpensslProbeError::RegexError)?;
    let mut buf = vec![0u8; 1024 * 1024]; // 1MB 缓冲区
    let mut total_read_count = 0;
    let mut version_key = String::new();

    while total_read_count < section_size {
        let read_count = (&buffer[section_offset + total_read_count..])
            .read(&mut buf)
            .map_err(OpensslProbeError::Io)?;
        if read_count == 0 {
            break;
        }

        if let Some(match_version) = rex.find(&buf[0..read_count]) {
            let version_str = std::str::from_utf8(match_version.as_bytes()).map_err(|_| {
                OpensslProbeError::Io(io::Error::new(
                    io::ErrorKind::InvalidData,
                    "Invalid UTF-8 sequence in matched version",
                ))
            })?;
            version_key = version_str.to_string();
            break;
        }

        total_read_count += read_count.saturating_sub(OPENSSL_VERSION_LEN);
    }

    if version_key.is_empty() {
        return Err(OpensslProbeError::VersionNotFound);
    }

    Ok(version_key.to_lowercase())
}

pub fn detect_openssl_version<P: AsRef<Path>>(so_path: P) -> Result<String, OpensslProbeError> {
    let so_path = so_path.as_ref();

    match detect_openssl_version_by_path(so_path) {
        Ok(version) => return Ok(version),
        Err(err) => {
            warn!(err = ?err, so_path = ?so_path.display(), "Failed to detect OpenSSL version");
        },
    }

    //  Get the path of libcrypto.so.3 from libssl.so.3
    let libcrypto_name = "libcrypto.so.3";
    let imd: Vec<String> = get_imp_needed(so_path).unwrap_or_else(|_| Vec::new());

    debug!(imported = ?imd, so_path = ?so_path.display(), "scanning imported libraries");

    let libcrypto_name = imd
        .into_iter()
        .find(|im| im.contains("libcrypto.so"))
        .unwrap_or(libcrypto_name.to_string());

    let so_path = so_path.parent().unwrap().join(libcrypto_name);
    info!("Try to detect imported libcrypto.so at {:?}", so_path);

    detect_openssl_version_by_path(&so_path)
}

pub fn get_imp_needed<P: AsRef<Path>>(so_path: P) -> Result<Vec<String>, OpensslProbeError> {
    let buffer = std::fs::read(so_path).map_err(OpensslProbeError::Io)?;
    let elf = goblin::elf::Elf::parse(&buffer).unwrap();
    let mut imported_needed = Vec::new();
    // Iterate over dynamic entries
    if let Some(dynamic) = elf.dynamic {
        // We need that strtab, and we have to do this one manually.
        // Get the strtab address
        let mut strtab_address = None;
        for dyn_ in &dynamic.dyns {
            if dyn_.d_tag == goblin::elf::dynamic::DT_STRTAB {
                strtab_address = Some(dyn_.d_val);
                break;
            }
        }
        if strtab_address.is_none() {
            return Ok(imported_needed);
        }
        let strtab_address = strtab_address.unwrap();
        // We're going to make a pretty safe assumption that strtab is all
        // in one section
        for section_header in &elf.section_headers {
            if section_header.sh_addr > 0
                && section_header.sh_addr <= strtab_address
                && section_header.sh_addr + section_header.sh_size > strtab_address
            {
                let start = section_header.sh_offset + (strtab_address - section_header.sh_addr);
                let size = section_header.sh_size - (start - section_header.sh_offset);
                let start = start as usize;
                let size = size as usize;
                let strtab_bytes = buffer.get(start..(start + size)).unwrap();
                let strtab = goblin::strtab::Strtab::new(strtab_bytes, 0);
                for dyn_ in dynamic.dyns {
                    if dyn_.d_tag == goblin::elf::dynamic::DT_NEEDED {
                        let so_file = &strtab[dyn_.d_val as usize];
                        imported_needed.push(so_file.to_string());
                    }
                }
                return Ok(imported_needed);
            }
        }
        // If we got here, we didn't return a vector (I think ;))
        panic!("Failed to get Dynamic strtab");
    }
    Ok(imported_needed)
}

```