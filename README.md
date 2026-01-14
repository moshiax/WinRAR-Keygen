A hook for implementing Winrar key generation using WASM from https://winrar.netlify.app binary
---
Slightly modified by me js implementation from https://github.com/danieldac1819/WinRAR-Keygen-Web
<pre>
const GF2_15_exp_tab = generateGF2_15_exp_tab();
const GF2_15_log_tab = generateGF2_15_log_tab();
const HEX_TAB = Array.from({length: 256}, (_, i) => i.toString(16).padStart(2, '0'));

function generateGF2_15_exp_tab() {
    const EXP_SIZE = 0x7fff; // 32767
    const exp_tab = new Uint32Array(EXP_SIZE);
    let p = 1;
    exp_tab[0] = 1;

    for (let i = 1; i < EXP_SIZE; i++) {
        p <<= 1;
        if (p > EXP_SIZE) {
            p ^= 0x8003; // Polynomial x^15 + x + 1
        }
        exp_tab[i] = p;
    }
    return exp_tab;
}

function generateGF2_15_log_tab() {
    const EXP_SIZE = 0x7fff; // 32767
    const LOG_SIZE = 0x7fff + 1; // 32768
    const log_tab = new Uint32Array(LOG_SIZE);
    log_tab[1] = 0;

    for (let i = 0; i < EXP_SIZE; i++) {
        log_tab[GF2_15_exp_tab[i]] = i;
    }
    log_tab[0] = 0; // Special case

    return log_tab;
}

function CRC32(r) { for (var a, o = [], c = 0; c < 256; c++) { a = c; for (var f = 0; f < 8; f++)a = 1 & a ? 3988292384 ^ a >>> 1 : a >>> 1; o[c] = a } for (var n = -1, t = 0; t < r.length; t++)n = n >>> 8 ^ o[255 & (n ^ r.charCodeAt(t))]; return (n) >>> 0 };

function Bytes_to_BigInt(b) {
    let ans = 0n;
    for (let i = 0; i < b.length; i++) ans |= BigInt(b[i].charCodeAt(0) & 0xff) << BigInt(i * 8);
    return ans;
}
function BigInt_to_Bytes(b, l) {
    let arr = Array(l);
    let _b = BigInt(b);
    for (let k = 0; k < l; k++) {
        arr[l - k - 1] = String.fromCharCode(Number(_b & 0xffn));
        _b >>= 8n;
    }
    return arr.join('');
}
function Bytes_to_Hex(b) {
    let ans = '';
    for (let i = 0; i < b.length; i++) ans += HEX_TAB[b[i].charCodeAt(0) & 0xff];
    return ans;
}
function GF_add(a, b) {
    return (a ^ b);
}

function GF_sub(a, b) {
    return (a ^ b);
}

function GF215_mul(s1, s2) {
    if (!s1 || !s2) return 0;
    return GF2_15_exp_tab[(GF2_15_log_tab[s1] + GF2_15_log_tab[s2]) % 0x7fff];
}
function GF_mul(a, b) {
    if (!a || !b) return 0n;
    const Ta = new Uint32Array(17);
    const Tb = new Uint32Array(17);
    const Tc = new Uint32Array(34);

    for (let k = 0; k < 17; k++) {
        Ta[k] = Number(a & 0x7fffn);
        Tb[k] = Number(b & 0x7fffn);
        a >>= 15n;
        b >>= 15n;
    }

    for (let k = 0; k < 17; k++) {
        for (let r = 0; r < 17; r++) Tc[k + r] ^= GF215_mul(Ta[k], Tb[r]);
    }

    for (let k = 33; k > 16; k--) {
        Tc[k - 17] ^= Tc[k];
        Tc[k - 14] ^= Tc[k];
        Tc[k] = 0;
    }

    let res = 0n;
    for (let k = 16; k >= 0; k--) res = (res << 15n) + BigInt(Tc[k]);
    return res;
}

function GF_inv(a) {
    if (a === 0n) return 0n;

    let ans = 1n;
    let base = a;

    for (let r = 0; r < 16; r++) {
        for (let i = 0; i < 15; i++) base = GF_mul(base, base);
        ans = GF_mul(ans, base);
    }

    const temp = BigInt(GF2_15_exp_tab[(0x7fff - GF2_15_log_tab[Number(GF_mul(ans, a) & 0x7fffn)]) % 0x7fff]);
    return GF_mul(ans, temp);
}

function GF_div(a, b) { return GF_mul(a, GF_inv(b)); }

function ECP_double(a) {
    if (a == 0n) { return a; }
    else {
        var _x, _y, m;
        var newx, newy;
        _x = a & (0x07fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffn);
        _y = (a >> (255n)) & (0x07fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffn);
        m = GF_add(GF_div(_y, _x), _x);
        newx = (GF_add(GF_mul(m, m), m));
        newy = (GF_add(GF_mul(GF_add(m, 1n), newx), GF_mul(_x, _x)));
        return (newx | (newy << (255n)));
    }
}
function ECP_add(a, b) {
    var a_x, a_y, b_x, b_y, m;
    var newx, newy;
    a_x = a & (0x07fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffn);
    a_y = (a >> (255n)) & (0x07fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffn);
    b_x = b & (0x07fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffn);
    b_y = (b >> (255n)) & (0x07fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffn);
    if (a == b) {
        return ECP_double(a);
    }
    else if ((a_x == 0n) && (a_y == 0n)) {
        return b;
    }
    else if ((b_x == 0n) && (b_y == 0n)) {
        return a;
    }
    else if ((a_x == 0n) && (b_x == 0n) && (a_y != 0n) && (b_y != 0n)) {
        return 0n;
    }
    else {
        m = GF_div(GF_add(a_y, b_y), GF_add(a_x, b_x));
        newx = (GF_add(GF_add(m, GF_mul(m, m)), GF_add(a_x, b_x)));
        newy = (GF_add(GF_mul(m, GF_add(a_x, newx)), GF_add(newx, a_y)));
        return (newx | (newy << (255n)));
    }
}

function ECP_smul(k, a) {
    if (k == 0n) { return 0n; }
    else {
        var _k, _a, ans;
        _k = k % (0x01026dd85081b82314691ced9bbec30547840e4bf72d8b5e0d258442bbcd31n);
        _a = a;
        ans = 0n;
        while (_k != 0n) {
            if ((_k & (1n)) != 0n) {
                ans = ECP_add(ans, _a);
            }
            _a = ECP_add(_a, _a);
            _k = (_k >> (1n));
        }
        return ans;
    }
}

function SHA1(msg) {
    function rotate_left(n, s) {
        return ((n << s) | (n >>> (32 - s)));
    };
    function cvt_hex(val) {
        var str = '';
        str += ((val >>> 28) & 0x0f).toString(16);
        str += ((val >>> 24) & 0x0f).toString(16);
        str += ((val >>> 20) & 0x0f).toString(16);
        str += ((val >>> 16) & 0x0f).toString(16);
        str += ((val >>> 12) & 0x0f).toString(16);
        str += ((val >>> 8) & 0x0f).toString(16);
        str += ((val >>> 4) & 0x0f).toString(16);
        str += (val & 0x0f).toString(16);
        return str;
    };
    var blockstart;
    var i, j;
    var W = new Array(80);
    var H0 = 0x067452301;
    var H1 = 0x0EFCDAB89;
    var H2 = 0x098BADCFE;
    var H3 = 0x010325476;
    var H4 = 0x0C3D2E1F0;
    var A, B, C, D, E;
    var temp;
    var msg_len = msg.length;
    var word_array = new Array();
    for (i = 0; i < msg_len - 3; i += 4) {
        j = msg.charCodeAt(i) << 24 | msg.charCodeAt(i + 1) << 16 | msg.charCodeAt(i + 2) << 8 | msg.charCodeAt(i + 3);
        word_array.push(j);
    }
    switch (msg_len % 4) {
        case 0:
            i = 0x080000000;
            break;
        case 1:
            i = msg.charCodeAt(msg_len - 1) << 24 | 0x0800000;
            break;
        case 2:
            i = msg.charCodeAt(msg_len - 2) << 24 | msg.charCodeAt(msg_len - 1) << 16 | 0x08000;
            break;
        case 3:
            i = msg.charCodeAt(msg_len - 3) << 24 | msg.charCodeAt(msg_len - 2) << 16 | msg.charCodeAt(msg_len - 1) << 8 | 0x080;
            break;
    }
    word_array.push(i);
    while ((word_array.length % 16) != 14) word_array.push(0);
    word_array.push(msg_len >>> 29);
    word_array.push((msg_len << 3) & 0x0ffffffff);
    for (blockstart = 0; blockstart < word_array.length; blockstart += 16) {
        for (i = 0; i < 16; i++) W[i] = word_array[blockstart + i];
        for (i = 16; i <= 79; i++) W[i] = rotate_left(W[i - 3] ^ W[i - 8] ^ W[i - 14] ^ W[i - 16], 1);
        A = H0;
        B = H1;
        C = H2;
        D = H3;
        E = H4;
        for (i = 0; i <= 19; i++) {
            temp = (rotate_left(A, 5) + ((B & C) | (~B & D)) + E + W[i] + 0x05A827999) & 0x0ffffffff;
            E = D;
            D = C;
            C = rotate_left(B, 30);
            B = A;
            A = temp;
        }
        for (i = 20; i <= 39; i++) {
            temp = (rotate_left(A, 5) + (B ^ C ^ D) + E + W[i] + 0x06ED9EBA1) & 0x0ffffffff;
            E = D;
            D = C;
            C = rotate_left(B, 30);
            B = A;
            A = temp;
        }
        for (i = 40; i <= 59; i++) {
            temp = (rotate_left(A, 5) + ((B & C) | (B & D) | (C & D)) + E + W[i] + 0x08F1BBCDC) & 0x0ffffffff;
            E = D;
            D = C;
            C = rotate_left(B, 30);
            B = A;
            A = temp;
        }
        for (i = 60; i <= 79; i++) {
            temp = (rotate_left(A, 5) + (B ^ C ^ D) + E + W[i] + 0x0CA62C1D6) & 0x0ffffffff;
            E = D;
            D = C;
            C = rotate_left(B, 30);
            B = A;
            A = temp;
        }
        H0 = (H0 + A) & 0x0ffffffff;
        H1 = (H1 + B) & 0x0ffffffff;
        H2 = (H2 + C) & 0x0ffffffff;
        H3 = (H3 + D) & 0x0ffffffff;
        H4 = (H4 + E) & 0x0ffffffff;
    }
    var temp = cvt_hex(H0) + cvt_hex(H1) + cvt_hex(H2) + cvt_hex(H3) + cvt_hex(H4);

    return temp.toLowerCase();
}

function Winrar_format_SHA1(data) {
    if (!data.length) return "\x81\xb7\x3e\xeb\x29\x53\x26\x50\xa3\xf4\x5e\xdc\xd5\xb9\x47\x68\x4c\x3b\xe4\xcd";

    const hex = SHA1(data);
    const out = new Array(20);
    for (let i = 0; i < 20; i += 4) {
        for (let j = 3; j >= 0; j--) {
            out[i + 3 - j] = String.fromCharCode(Number("0x" + hex.substr(2 * (i + j), 2)));
        }
    }
    return out.join('');
}

function Winrar_get_privkey(data) {
    var d = Winrar_format_SHA1(data);
    var pri_key = "";

    for (var i = 1; i <= 0x0f; i++) {
        pri_key += Winrar_format_SHA1(
            String.fromCharCode(i) + "\x00\x00\x00" + d
        ).substring(0, 2);
    }

    return pri_key;
}

function Winrar_padded_hash(data) {
    return Winrar_format_SHA1(data) + "\x43\x8d\xfd\x0f\x7c\x3c\xe3\xb4\xd1\x1b";
}

function Winrar_get_pubkey(privkey) {
    var Base_Point = 0x0106d197457e485be7873b5ef2224b5a683ca02a73753079c43412318997c98d3d6fdcbc6a27acee0cc2996e0096ae74feb1acf220a2341b898b549440297b8ccn;
    var privkey_num = 0n;
    var pkG = 0n;
    var pkG_x = 0n;
    var pkG_y = 0n;
    var pubk_1 = 0n;
    var pubk_2 = 0n;
    privkey_num = Bytes_to_BigInt(privkey);
    pkG = ECP_smul(privkey_num, Base_Point);
    pkG_x = pkG & (0x07fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffn);
    pkG_y = (pkG >> (255n)) & (0x07fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffn);
    pubk_1 = (2n) * (pkG_x);
    pubk_2 = GF_div(pkG_y, pkG_x) & (1n);
    return pubk_1 + pubk_2;
}

function Winrar_sign(data) {
    var Base_Point = 0x0106d197457e485be7873b5ef2224b5a683ca02a73753079c43412318997c98d3d6fdcbc6a27acee0cc2996e0096ae74feb1acf220a2341b898b549440297b8ccn;
    var winrar_priv_k = 0x059fe6abcca90bdb95f0105271fa85fb9f11f467450c1ae9044b7fd61d65en;
    var order_n = 0x01026dd85081b82314691ced9bbec30547840e4bf72d8b5e0d258442bbcd31n;
    var random = 0n;
    var Random_a = new Uint32Array(2);
    var sign_r = 0n;
    var sign_s = 0n;
    crypto.getRandomValues(Random_a);
    random = ((random + BigInt(Random_a[0])) << (32n));
    random = ((random + BigInt(Random_a[1])));
    random = random & (0x0ffn);
    sign_r = ((order_n << (512n)) + (ECP_smul(random, Base_Point) & (0x07fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffn)) + Bytes_to_BigInt(Winrar_padded_hash(data))) % order_n;
    sign_s = ((order_n << (512n)) + random - (winrar_priv_k * sign_r)) % order_n;
    return [sign_r, sign_s];
}

function Winrar_KeyGen(USER, LIC) {
    const temp = BigInt_to_Bytes(Winrar_get_pubkey(Winrar_get_privkey(USER)), 32);
    const data3 = Bytes_to_Hex("\x60" + temp.substring(0, 24));
    const data0 = Bytes_to_Hex(BigInt_to_Bytes(Winrar_get_pubkey(Winrar_get_privkey(data3)), 32));

    let data1_r, data1_s;
    do [data1_r, data1_s] = Winrar_sign(LIC); while (data1_r >= 1n << 239n || data1_s >= 1n << 239n);
    const data1 = "60" + Bytes_to_Hex(BigInt_to_Bytes(data1_s, 30)) + Bytes_to_Hex(BigInt_to_Bytes(data1_r, 30));

    let data2_r, data2_s;
    do [data2_r, data2_s] = Winrar_sign(USER + data0); while (data2_r >= 1n << 239n || data2_s >= 1n << 239n);
    const data2 = "60" + Bytes_to_Hex(BigInt_to_Bytes(data2_s, 30)) + Bytes_to_Hex(BigInt_to_Bytes(data2_r, 30));

    let data = data0 + data1 + data2 + data3;
    data = "6412212250" + data + CRC32(LIC + USER + data).toString(10).padStart(10, "0");

    const RAR_UID = Bytes_to_Hex(temp.substring(24, 32)) + data0.substring(0, 4);

    let KEY_STR = `RAR registration data\n${USER}\n${LIC}\nUID=${RAR_UID}\n`;
    for (let i = 0; i < 7; i++) KEY_STR += data.substring(i * 54, (i + 1) * 54) + "\n";

    return KEY_STR;
}

      
let user = "Microsoft Corporation";
let lic = "Unlimited Perpetual License";
const key = Winrar_KeyGen(user, lic);
console.log(key);
</pre>