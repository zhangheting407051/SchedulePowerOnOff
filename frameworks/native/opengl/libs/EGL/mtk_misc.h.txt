#define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]))
static const char *rgb888_support_list[] = \
{
    "com.mediatek.camera"    // camera PIP mode
    ,"com.android.systemui"
};

inline int MTK_CheckAppName(const char* acAppName)
{
    char appName[128] = {0};
    char procPath[128] = {0};
    long pid = getpid();

    snprintf(procPath, 128, "/proc/%ld/cmdline", pid);
    FILE * file = fopen(procPath, "r");
    if (file)
    {
        fgets(appName, sizeof(appName)-1, file);
        fclose(file);
    }
    //ALOGD("appName: %s, acAppNAme: %s", appName, acAppName);
    if (strstr(appName, acAppName))
    {
        /// ALOGD("1");
        return 1;
    }
    /// ALOGD("0");
    return 0;
}

int MTK_isRGB888Supported(){
    int i;
    char buf[PROPERTY_VALUE_MAX];
    int ret = 0;

    if(property_get_int32("debug.egl.rgb888_enable", 0)==1) return 1;
    if(property_get_int32("ro.mtk_gmo_ram_optimize", 0)==1) ret = 1;

    if(ret==1){
        property_get("ro.product.device", buf, "");
        if(!strncmp(buf, "k35", 3) ||
                !strncmp(buf, "k37", 3) ||
                !strncmp(buf, "k53", 3) ||
                !strncmp(buf, "k55", 3) ||
                !strncmp(buf, "k50", 3) ||
                !strncmp(buf, "k57", 3))
        {
            for(i=0; i<(int)ARRAY_SIZE(rgb888_support_list) ;i++)
            {
                if(MTK_CheckAppName(rgb888_support_list[i])) return 1;
            }
        }
        ret = 0;
    }
    return ret;
}

