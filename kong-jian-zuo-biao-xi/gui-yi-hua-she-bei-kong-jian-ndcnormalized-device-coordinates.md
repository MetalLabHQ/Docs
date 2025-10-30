# 归一化设备空间 NDC(Normalized Device Coordinates)

通过 **视锥体挤压 Frustum Squeeze** 中进行 **透视除法 Perspective Division** ，将 [cai-jian-kong-jian-clip-space.md](cai-jian-kong-jian-clip-space.md "mention") 坐标转换至 NDC

\
Metal 中，NDC的坐标范围通常是x轴和y轴在\[-1, 1]之间，z轴在\[0, 1]之间
