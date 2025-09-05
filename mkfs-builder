#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <inttypes.h>
#include <errno.h>
#include <time.h>
#include <assert.h>

#define BS 4096u               // block size
#define INODE_SIZE 128u
#define ROOT_INO 1u

uint64_t g_random_seed = 0; // seed from CLI if needed

// ========================== STRUCTS ==========================
#pragma pack(push,1)
typedef struct {
    uint32_t magic;
    uint32_t version;
    uint32_t block_size;
    uint64_t total_blocks;
    uint64_t inode_count;
    uint64_t inode_bitmap_start;
    uint64_t inode_bitmap_blocks;
    uint64_t data_bitmap_start;
    uint64_t data_bitmap_blocks;
    uint64_t inode_table_start;
    uint64_t inode_table_blocks;
    uint64_t data_region_start;
    uint64_t data_region_blocks;
    uint64_t root_inode;
    uint64_t mtime_epoch;
    uint32_t flags;
    uint32_t checksum; // MUST stay at the end
} superblock_t;
#pragma pack(pop)
_Static_assert(sizeof(superblock_t) <= BS, "superblock must fit in one block");

#pragma pack(push,1)
typedef struct {
    uint16_t mode;        // 2
    uint16_t links;       // 2
    uint32_t uid;         // 4
    uint32_t gid;         // 4
    uint64_t size_bytes;  // 8
    uint64_t atime;       // 8
    uint64_t mtime;       // 8
    uint64_t ctime;       // 8
    uint32_t direct[10];  // 40
    uint8_t pad[36];      // pad to reach 120 before CRC
    uint64_t inode_crc;   // 8
} inode_t;
#pragma pack(pop)
_Static_assert(sizeof(inode_t) == INODE_SIZE, "inode size mismatch");

#pragma pack(push,1)
typedef struct {
    uint32_t inode_no; // 4
    uint8_t  type;     // 1
    char name[58];     // 58
    uint8_t checksum;  // 1
} dirent64_t;
#pragma pack(pop)
_Static_assert(sizeof(dirent64_t) == 64, "dirent size mismatch");

// ========================== CRC32 HELPERS ==========================
uint32_t CRC32_TAB[256];
void crc32_init(void){
    for (uint32_t i=0;i<256;i++){
        uint32_t c=i;
        for(int j=0;j<8;j++) c = (c&1)?(0xEDB88320u^(c>>1)):(c>>1);
        CRC32_TAB[i]=c;
    }
}
uint32_t crc32(const void* data, size_t n){
    const uint8_t* p=(const uint8_t*)data; uint32_t c=0xFFFFFFFFu;
    for(size_t i=0;i<n;i++) c = CRC32_TAB[(c^p[i])&0xFF] ^ (c>>8);
    return c ^ 0xFFFFFFFFu;
}

// ========================== FINALIZE HELPERS ==========================
static uint32_t superblock_crc_finalize(superblock_t *sb) {
    sb->checksum = 0;
    uint32_t s = crc32((void *) sb, BS - 4);
    sb->checksum = s;
    return s;
}

void inode_crc_finalize(inode_t* ino){
    uint8_t tmp[INODE_SIZE]; memcpy(tmp, ino, INODE_SIZE);
    memset(&tmp[120], 0, 8); // zero crc area
    uint32_t c = crc32(tmp, 120);
    ino->inode_crc = (uint64_t)c; // low 4 bytes carry the crc
}

void dirent_checksum_finalize(dirent64_t* de) {
    const uint8_t* p = (const uint8_t*)de;
    uint8_t x = 0;
    for (int i = 0; i < 63; i++) x ^= p[i];
    de->checksum = x;
}

static void set_bit(uint8_t* bmap, uint64_t idx){
    bmap[idx>>3] |= (uint8_t)(1u << (idx & 7u));
}

// ========================== MAIN ==========================
int main(int argc, char** argv) {
    crc32_init();
    const char* image_path = NULL;
    long size_kib = -1;
    long inode_cnt = -1;

    for (int i=1;i<argc;i++){
        const char* a = argv[i];
        if (strcmp(a,"--image")==0 && i+1<argc){ image_path = argv[++i]; }
        else if (strcmp(a,"--size-kib")==0 && i+1<argc){ size_kib = strtol(argv[++i], NULL, 10); }
        else if (strcmp(a,"--inodes")==0 && i+1<argc){ inode_cnt = strtol(argv[++i], NULL, 10); }
        else { return 2; }
    }
    if (!image_path || size_kib<180 || size_kib>4096 || (size_kib%4)!=0 || inode_cnt<128 || inode_cnt>512) return 2;

    uint64_t total_blocks = (uint64_t)size_kib / 4u;
    uint64_t itbl_blocks   = ((uint64_t)inode_cnt + 31) / 32;
    uint64_t ib_start = 1, db_start = 2, itbl_start = 3;
    uint64_t dreg_start = itbl_start + itbl_blocks;
    if (total_blocks <= dreg_start) return 2;
    uint64_t dreg_blocks = total_blocks - dreg_start;

    size_t img_bytes = (size_t)(total_blocks * BS);
    uint8_t* img = (uint8_t*)calloc(1, img_bytes);
    if (!img) return 2;

    superblock_t sb; memset(&sb, 0, sizeof(sb));
    sb.magic = 0x4D565346u; sb.version=1; sb.block_size = BS;
    sb.total_blocks = total_blocks;
    sb.inode_count  = (uint64_t)inode_cnt;
    sb.inode_bitmap_start = ib_start; sb.inode_bitmap_blocks=1;
    sb.data_bitmap_start  = db_start; sb.data_bitmap_blocks = 1;
    sb.inode_table_start  = itbl_start; sb.inode_table_blocks = itbl_blocks;
    sb.data_region_start  = dreg_start; sb.data_region_blocks = dreg_blocks;
    sb.root_inode = ROOT_INO;
    sb.mtime_epoch = (uint64_t)time(NULL);
    sb.flags = 0;
    superblock_crc_finalize(&sb);
    memcpy(img + 0*BS, &sb, sizeof(sb));

    uint8_t* ibmap = img + (size_t)ib_start*BS;
    uint8_t* dbmap = img + (size_t)db_start*BS;
    set_bit(ibmap, 0);
    set_bit(dbmap, 0);

    inode_t root; memset(&root, 0, sizeof(root));
    root.mode = 040000; root.links = 2; root.uid = 0; root.gid = 0;
    root.size_bytes = 2u * sizeof(dirent64_t);
    root.atime = root.mtime = root.ctime = (uint64_t)time(NULL);
    root.direct[0] = (uint32_t)dreg_start;
    inode_crc_finalize(&root);
    memcpy(img + (size_t)itbl_start*BS + 0*INODE_SIZE, &root, sizeof(root));

    dirent64_t dot={0}, dotdot={0};
    dot.inode_no = ROOT_INO; dot.type = 2; strncpy(dot.name, ".", 57); dirent_checksum_finalize(&dot);
    dotdot.inode_no = ROOT_INO; dotdot.type=2; strncpy(dotdot.name,"..",57); dirent_checksum_finalize(&dotdot);
    memcpy(img + (size_t)dreg_start*BS + 0*sizeof(dirent64_t), &dot, sizeof(dot));
    memcpy(img + (size_t)dreg_start*BS + 1*sizeof(dirent64_t), &dotdot, sizeof(dotdot));

    FILE* f = fopen(image_path, "wb");
    if (!f){ free(img); return 2; }
    if (fwrite(img,1,img_bytes,f)!=img_bytes){ fclose(f); free(img); return 2; }
    fclose(f); free(img);

    // **Confirmation message added here**:
    printf("File system image '%s' has been successfully created.\n", image_path);

    return 0;
}
