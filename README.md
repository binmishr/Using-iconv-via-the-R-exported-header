# Using-iconv-via-the-R-exported-header

Introduction

Character encodings can be tricky and frustrating to deal with. Several newer languages such as Go or Julia default to native UTF-8 on all platforms, greatly facilitating and easing use of characters with languages other than English on all common platforms. With R we are not quite as lucky. UTF-8 is mostly working as desired on “operating systems with an x” but sadly, there are still a lot of Windows users out there for whom native UTF-8 is not quite in reach. A very detailed discussion of the issues involved was provided last summer on the R Developers Blog in this post.

More recently, another blog post. The useful idea presented in the post is to rely on the (public) header R_ext/Riconv.h which then transparently passes on to the iconv library R itself uses. (Strictly speaking this is an optional feature, see capabilities("iconv") to check your build of R.)

In order to test this, , we could toss the file at the accessiable Windows builders for tests (given that we do not have a physical Windows machine around). Together with an input file encoded in windows-1252 (taken from the uchardet CRAN package wrapping Mozilla’s uchardet library) we can then read and convert text in these ‘foreign’ encoding:

win1252file <- system.file("rawdata", "windows-1252.txt", package="RcppIconvExample")
win1252txt <- RcppIconvExample::read_file(win1252file, "windows-1252")
cat(win1252txt)

L’œuf de volaille est un produit agricole servant d'ingrédient entrant dans la
composition de nombreux plats, dans de nombreuses cultures gastronomiques du
monde.

Our implementation of read_file() follows. It refactors the two functions in the blog post into a single function with an optional encoding argument:

// cf https://fishandwhistle.net/post/2021/using-rs-cross-platform-iconv-wrapper-from-cpp11
std::string read_file(std::string filename, std::string encoding = "") {
    const int len = 2048;
    char buffer[len/2];

    std::ifstream file;
    file.open(filename, std::ifstream::in | std::ifstream::binary);

    file.read(buffer, len/2);
    size_t n_read = file.gcount();
    file.close();

    if (encoding == "") {       // no encoding given so return 'as is'
        return std::string(buffer, n_read);
    }

    std::string str_source(buffer, n_read);

    void* iconv_handle = Riconv_open("UTF-8", encoding.c_str());
    if (iconv_handle == ((void*) -1)) {
        Rcpp::stop("Can't convert from '%s' to 'UTF-8'", encoding.c_str());
    }

    const char* in_buffer = str_source.c_str();
    char utf8_buffer[len];
    char* utf8_buffer_mut = utf8_buffer;
    size_t in_bytes_left = n_read;
    size_t out_bytes_left = len;

    size_t result = Riconv(iconv_handle, &in_buffer, &in_bytes_left, &utf8_buffer_mut, &out_bytes_left);
    Riconv_close(iconv_handle);

    if (result == ((size_t) -1) || (in_bytes_left != 0)) {
        Rcpp::stop("Failed to convert file contents to UTF-8");
    }

    return std::string(utf8_buffer, len - out_bytes_left);
}

The entire function body is plain C++ code in a basic C++1998 standard, calls the C API of R to access iconv if a conversion is selected, and relies on Rcpp for the convenience of automating the interface and translating strings to SEXP objects and back.
