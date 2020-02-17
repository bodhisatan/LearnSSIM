# FFmpeg对SSIM的实现

源码：tiny_ssim.c

链接：https://github.com/wangwei1237/video_codec_evaluation/blob/master/test/test_ssim.cpp

代码前面叭叭叭说了一大堆，有用的只有两句，一是告诉了我们输入格式：两个YV12（YUV420P）格式视频文件，二是告诉我们为了提升速度，代码没有用论文里的高斯卷积核作加权平均，而是用了8x8有重叠的像素块求和的方法。

读源码，先从main函数读起

```c
	FILE *f[2];
	uint8_t *buf[2], *plane[2][3];
    int *temp;
    uint64_t ssd[3] = {0, 0, 0};
    double ssim[3] = {0, 0, 0};
    int frame_size, w, h;
    int frames, seek;
    int i;

    if (argc < 4 || 2 != sscanf(argv[3], "%dx%d", &w, &h))
    {
        printf("tiny_ssim <file1.yuv> <file2.yuv> <width>x<height> [<seek>]\n");
        return -1;
    }

    f[0] = fopen(argv[1], "rb");
    f[1] = fopen(argv[2], "rb");
    sscanf(argv[3], "%dx%d", &w, &h);

    if (w <= 0 || h <= 0 || w * (int64_t)h >= INT_MAX / 3 || 2LL * w + 12 >= INT_MAX / sizeof(*temp))
    {
        fprintf(stderr, "Dimensions are too large, or invalid\n");
        return -2;
    }
```

先进行了变量定义和初始化，规定了程序运行格式，读入了两个文件f[0]和f[1]，以及视频长宽w，h。

```c
frame_size = w * h * 3LL / 2;
for (i = 0; i < 2; i++)
{
        buf[i] = (uint8_t *)malloc(frame_size);
        plane[i][0] = buf[i];
        plane[i][1] = plane[i][0] + w * h;
        plane[i][2] = plane[i][1] + w * h / 4;
}
```

这句话就要联系我上篇wiki中关于YUV420P的知识，视频文件在计算机中被读取/计算的时候，数据是以字节流的形式存储的，frame_size就是一帧视频占内存的大小。Y占w\*h字节，U和V都是占1/4\*w\*h个字节，合起来就是1.5\*w\*h个字节。

一帧数据读进内存之后，分离出Y，U，V三个分量。因为YUV420P是planar格式存储的，先Y然后U最后V，所以只需要读取的时候加一个内存偏移量就能分别读到这三个分量。plane\[i\]\[0\]存储Y分量信息，plane\[i\]\[1\]存储U分量信息，plane\[i\]\[2\]存储V分量信息。i取0和1，这是用来区分两个输入文件的。

接下来就开始了逐帧计算：

```c
    for (frames = 0;; frames++)
    {
        uint64_t ssd_one[3]; 
        double ssim_one[3]; 
        if (fread(buf[0], frame_size, 1, f[0]) != 1)
            break;
        if (fread(buf[1], frame_size, 1, f[1]) != 1)
            break;
        for (i = 0; i < 3; i++)
        {
            ssd_one[i] = ssd_plane(plane[0][i], plane[1][i], w * h >> 2 * !!i);
            ssim_one[i] = ssim_plane(plane[0][i], w >> !!i,
                                     plane[1][i], w >> !!i,
                                     w >> !!i, h >> !!i, temp, NULL);
            ssd[i] += ssd_one[i];
            ssim[i] += ssim_one[i];
        }

        printf("Frame %d | ", frames);
        print_results(ssd_one, ssim_one, 1, w, h);
        printf("                \r");
        fflush(stdout);
    }
```

ssd_one和ssime_one代表一帧数据的ssd与ssim结果，ssd的含义在前一篇wiki中也解释过了。他们都是大小为3的数组，因为Y，U，V三个数据分量需要分别计算存储。接下来两行fread，读取数据送入buf中，但因为前文的代码里Y分量的起始存储位置就是从buf开始的，所以plane数组就自动的获得了数据。不得不感叹代码的精妙！

接下来是一个i从0到3的循环，i=0时比较Y，i=1时比较U，i=2时比较V，ssd_plane是按plane计算ssd，ssim_plane是按照plane计算ssim，ssd_plane代码如下：

```c
uint64_t ssd_plane(const uint8_t *pix1, const uint8_t *pix2, int size)
{
    uint64_t ssd = 0;
    int i;
    for (i = 0; i < size; i++)
    {
        int d = pix1[i] - pix2[i];
        ssd += d * d;
    }
    return ssd;
}
```

先分析上文调用的语句``ssd_one[i] = ssd_plane(plane[0][i], plane[1][i], w * h >> 2 * !!i);`` ``plane[0][i]``表示原视频的像素信息起始地址，``plane[1][i]``表示重建后视频的像素信息起始地址，``w*h >> 2*!!i``表示这段信息的大小，这里的``!!i``就有点让人费解，仔细一想，再次不得不叹服这段代码的巧妙。``!!i``在i=0时值是0，大于0时值是1，所以这就巧妙地得到了Y向量信息占内存的大小（w*h），U/V向量占内存的大小（w \*h >> 2）。

ssd的计算分析完了，现在开始分析ssim的计算，先看一下所调函数ssim_plane的代码

```c
float ssim_plane(
    pixel *pix1, intptr_t stride1,
    pixel *pix2, intptr_t stride2,
    int width, int height, void *buf, int *cnt)
{
    int z = 0;
    int x, y;
    float ssim = 0.0;
    
    int(*sum0)[4] = (int(*)[4])buf; 
    int(*sum1)[4] = sum0 + (width >> 2) + 3;
    width >>= 2;
    height >>= 2; 
    for (y = 1; y < height; y++)
    {
        for (; z <= y; z++)
        {
            // FFSWAP( (int (*)[4]), sum0, sum1 );
            int(*tmp)[4] = sum0;
            sum0 = sum1;
            sum1 = tmp;

            for (x = 0; x < width; x += 2)
                ssim_4x4x2_core(&pix1[4 * (x + z * stride1)], stride1, &pix2[4 * (x + z * stride2)], stride2, &sum0[x]);
        }
        for (x = 0; x < width - 1; x += 4)
            ssim += ssim_end4(sum0 + x, sum1 + x, FFMIN(4, width - x - 1));
    }
    //     *cnt = (height-1) * (width-1);
    return ssim / ((height - 1) * (width - 1));
}

```

上文调用它的语句是``ssim_one[i] = ssim_plane(plane[0][i], w >> !!i, plane[1][i], w >> !!i, w >> !!i, h >> !!i, temp, NULL);``对于U和V向量来说，他们的内存占比是四分之一w\*h，那么在填写width和height参数的时候就要分别取w和h的二分之一。

然后看ssim_plane函数体，这个函数是按照4x4的块对像素进行处理的，使用sum1保存上一行块的“信息”，sum0保存当前一行块的“信息”。sum0、sum1是一个数组指针，其中存储了一个4元素数组的地址，换句话说，sum0、sum1中每一个元素对应一个4x4块的信息（该信息包含4个元素）。**4个元素中，[0]代表原始像素之和，[1]代表重建像素之和，[2]代表原始像素平方之和+重建像素平方之和，[3]代表原始像素*重建像素的值的和**。然后width和height分别右移两位（÷4），因为这一步的计算是以4*4的像素块为基本单位的。然后进入循环，看到这句话，``for (; z <= y; z++)``在这个函数开头，定义了z=0，也就意味着这个循环体里的语句在第一次执行时会执行两次，其他时候就会执行一次（妙啊），为什么要执行两次呢？因为sum0存储的是一行里4x4块的信息，sum1里存着上一行里的4x4块的信息，在下文的ssim_4x4x2_core运算中，是要将这两行合起来，计算**有重叠8x8像素块的信息**。在这一步循环中``for (x = 0; x < width; x += 2)``，x每次加2，也就是每次前进两个像素，然后在ssim_4x4x2_core中计算两个4x4的像素块，也就是说，在一行中，这些4x4的像素块都是两两重叠的。

接下来进入这个循环：``for (x = 0; x < width - 1; x += 4)``，x每次+4，步长为4，每次跳过4*4个int，进入ssim_end4:

```c
static float ssim_end4(int sum0[5][4], int sum1[5][4], int width)
{
    float ssim = 0.0;
    int i;

    for (i = 0; i < width; i++)
        ssim += ssim_end1(sum0[i][0] + sum0[i + 1][0] + sum1[i][0] + sum1[i + 1][0],
                          sum0[i][1] + sum0[i + 1][1] + sum1[i][1] + sum1[i + 1][1],
                          sum0[i][2] + sum0[i + 1][2] + sum1[i][2] + sum1[i + 1][2],
                          sum0[i][3] + sum0[i + 1][3] + sum1[i][3] + sum1[i + 1][3]);
    return ssim;
}
```

一开始的时候，我因为指针不熟悉，有这样一个疑问，在ssim_4x4x2_core中，每次计算的是``sums[0][0~3]``和``sums[1][0~3]``，每次都计算一样的值吗，循环变量x的变化又体现在哪里呢？以及，在ssim_end4中，参与运算的是``sum0[0~width][0~3]``啊，在ssim_4x4x2_core里，只计算了``sum[0~1][0~3]``啊，很多在这步需要参与运算的值，在之前都没计算过啊。一番查资料，才知道：

ssim_4x4x2_core里的sums和ssim_end4里的sum意义并不完全相同，和ssim_plane里的sum意义也不相同。sum0和sum1是一个``int[*][4]``，也就是指向int[4]数组的指针，C/C++里，二位数组在传参的时候，第一个维度是可以不写的（写是为了起提示作用），也就是说，ssim_4x4x2_core里的``int sums[2][4]``和ssim_end4里的``int sum0[5][4], int sum1[5][4]``在数据类型上并没有什么不同，都是``int[*][4]``，在for语句中，循环变量x第一次+2，第二次+4，因为x变了，指针的首地址也就变了，所以每次在ssim_4x4x2_core里计算的sums虽然看似每次都是计算``sums[0~1][0~3]``，实际上每次计算的都是不同的4x4块的信息。理解了这个，对应的ssim_end4里的变量sum0/sum1也就可以理解了。

然后，就到了最终汇总结果，通过sums求ssim的部分：ssim_end1:

```c
static float ssim_end1(int s1, int s2, int ss, int s12)
{
/* Maximum value for 10-bit is: ss*64 = (2^10-1)^2*16*4*64 = 4286582784, which will overflow in some cases.
 * s1*s1, s2*s2, and s1*s2 also obtain this value for edge cases: ((2^10-1)*16*4)^2 = 4286582784.
 * Maximum value for 9-bit is: ss*64 = (2^9-1)^2*16*4*64 = 1069551616, which will not overflow. */
#if BIT_DEPTH > 9
    typedef float type;
    static const float ssim_c1 = .01 * .01 * PIXEL_MAX * PIXEL_MAX * 64;
    static const float ssim_c2 = .03 * .03 * PIXEL_MAX * PIXEL_MAX * 64 * 63;
#else
    typedef int type;
    // k1=0.01, k2=0.03
    static const int ssim_c1 = (int)(.01 * .01 * PIXEL_MAX * PIXEL_MAX * 64 + .5);
    static const int ssim_c2 = (int)(.03 * .03 * PIXEL_MAX * PIXEL_MAX * 64 * 63 + .5);
#endif
    type fs1 = s1;
    type fs2 = s2;
    type fss = ss;
    type fs12 = s12;
    type vars = fss * 64 - fs1 * fs1 - fs2 * fs2;
    type covar = fs12 * 64 - fs1 * fs2;
    return (float)(2 * fs1 * fs2 + ssim_c1) * (float)(2 * covar + ssim_c2) / ((float)(fs1 * fs1 + fs2 * fs2 + ssim_c1) * (float)(vars + ssim_c2));
}
```

由上文分析我们可以得知：
$$
s1=\sum\sum a(i,j)=fs1\\
s2=\sum\sum b(i,j)=fs2\\
ss=\sum\sum [b(i,j) ^ 2 + a(i,j) ^ 2]=fss\\
s12=\sum\sum a(i,j)*b(i,j)=fs12\\
$$
而我们从上篇Wiki可以知道SSIM的化简公式：
$$
SSIM(a,b)=\frac{(2\mu_a\mu_b+C_1)(2\sigma_{ab}+C_2)}{(\mu_a^2+\mu_b^2+C_1)(\sigma_a^2+\sigma_b^2+C_2)}
$$
需要将这个表达式用fs1,fs2,fss,fs12变量表示。先计算均值，方差，协方差：
$$
\mu_a=\frac{1}{64}fs1\\
\mu_b=\frac{1}{64}fs2\\
\sigma_a^2+\sigma_b^2=\frac{1}{63}(\sum_{i,j}(a(i,j)-\mu_a)^2 + \sum_{i,j}(b(i,j)-\mu_b)^2)\\
=\frac{1}{63}\sum_{i,j}(a(i,j)^2+\mu_a^2-2*a(i,j)*\mu_a+b(i,j)^2+\mu_b^2-2*b(i,j)*\mu_b)\\
=\frac{1}{63}(\sum_{i,j}(a(i,j)^2+b(i,j)^2)-2(\mu_a\sum_{i,j}a(i,j)+\mu_b\sum_{i,j}b(i,j))+\sum_{i,j}(\mu_a^2+\mu_b^2))\\
=\frac{1}{63}(fss-2*\frac{1}{64}(fs1^2+fs2^2)+\frac{1}{64}(fs1^2+fs2^2)\\
=\frac{1}{63}(fss-\frac{1}{64}(fs1^2+fs2^2))\\
=\frac{1}{63}\frac{1}{64}(64fss-fs1^2-fs2^2)\\
\sigma_{ab}=\frac{1}{63}\sum_{i,j}((a(i,j)-\mu_a)(b(i,j)-\mu_b))\\
=\frac{1}{63}\sum_{i,j}(a(i,j)*b(i,j)-a(i,j)\mu_b-b(i,j)\mu_a+\mu_a*\mu_b)\\
=\frac{1}{63}(fs12-fs1*\mu_b-fs2*\mu_a+fs1*\mu_b)\\
=\frac{1}{63}(fs12-\frac{1}{64}fs1*fs2)\\
=\frac{1}{63}\frac{1}{64}(64fs12-fs1*fs2)
$$
带入SSIM公式中，分子分母左边的项约去1/(64\*64)，右边的项约去1/(63\*64)：
$$
SSIM(a,b)=\frac{(2fs1*fs2+64^2C_1)(2*64fs12-2fs1fs2+63*64C_2)}{(fs1^2+fs2^2+64^2C_1)(64fss-s1^2-s2^2+63*64C_2)}
$$
这就是代码里的：

```c
(float)(2 * fs1 * fs2 + ssim_c1) * (float)(2 * covar + ssim_c2) / ((float)(fs1 * fs1 + fs2 * fs2 + ssim_c1) * (float)(vars + ssim_c2));
```

不过，在这段代码中，ssim_c1理应等于``0.01 * 0.01 * 255 * 255 * 64 * 64 + 0.5``**作者似乎少乘一个64**。

以上，就是FFmpeg对SSIM的实现，读前人的代码，一开始读不懂，但越读越有深意，越读越能感受到作者的智慧与推敲的匠心。



