# 计算ERA5物理量与海冰之间的相关系数 - 插值，计算，与绘图

## 数据
在此示例中，我们使用了两组数据，分别是2m 温度和海冰浓度。2m温度来自ERA5 monthly averaged data on single levels from 1979 to present中2013年的数据（https://cds.climate.copernicus.eu/cdsapp#!/dataset/reanalysis-era5-single-levels-monthly-means?tab=form)，数据精度为0.25度，每月一个时间层，故数据格式为(1440,721,12)。
```
t2m_era=double(ncread('new_temp.nc','t2m'));
lon_era=double(ncread('new_temp.nc','longitude'));
lat_era=double(ncread('new_temp.nc','latitude'));
[lat_era,lon_era]=meshgrid(lat_era,lon_era);
```
海冰浓度数据来自NSIDC 2013年的北极海冰数据，原始数据时间步幅为两天一个数据点，为了保证两组数据时间精度一致，将其重新计算为月平均，故数据格式为(304,448,12)。
```
load('si_year');
```
## 插值
由于era5和NSIDC数据栅格不同，我们应对其进行插值。此处我们以NSIDC的数据为目标栅格，将ERA5的数据插值到NSIDC的栅格上：
```
t2m_si=NaN(size(si_year)); %被插值到NSIDC栅格的era5数据

%插值过程
for i=1:size(si_year,3);
    tic
    si_here=si_year(:,:,i);
    
    t2m_here=t2m_era(:,:,i);
    F=scatteredInterpolant(lon_era(:),lat_era(:),t2m_here(:),...
        'linear','linear');
    t2m_si(:,:,i)=reshape(F(lon_si(:),lat_si(:)),size(t2m_si,1),size(t2m_si,2));
    toc
end
```
## 计算相关系数
此时，我们得到了被插值到NSIDC栅格伤的era5数据。这样，t2m_si和si_year的栅格就完全相同了，可以进行点对点地计算相关系数。

```
cor_used=NaN(304,448); %相关系数
p_used=NaN(304,448);   %显著性

for i=1:size(lon_si,1);
    for j=1:size(lon_si,2);
        si_here=squeeze(si_year(i,j,:));
        t2m_here=squeeze(t2m_si(i,j,:));
        if nansum(si_here>0)>0;
            t2m_here(isnan(si_here))=[];
            si_here(isnan(si_here))=[];
            [c,p]=corr(si_here(:),t2m_here(:));
            cor_used(i,j)=c;
            p_used(i,j)=p;
        else
            cor_used(i,j)=nan;
            p_used(i,j)=nan;
        end
    end
end
```
## 绘图
```
m_proj('stereographic','lat',90,'long',30,'radius',60);
m_pcolor(lon_si,lat_si,cor_used);
shading flat
colormap(colorbwr);
colorbar
m_coast('linewidth',2,'color','k');
m_grid('ytick',[],'fontsize',14);
```
![Image text](https://github.com/LiHuaVUP/Cor_ERA5_NSIDC/blob/main/cor_icet2m.png)
## 其他问题
有一些问题需要注意：
1. 在这里，我只是用了2013年的数据。但实际上，计算相关系数应该需要尽可能长的数据，如10-20年。
2. 直接使用原始数据计算相关系数是不合适的，因为地球科学中的物理量大都有季节性（如夏季降水多，冬季降水少），直接使用原始数据计算的话计算出的相关系数可能有很大一部分来自这种季节性。因此，在计算相关系数时，通常要用移除了季节性/气候态（seasonality/climatology）之后的差异值（anomaly）来计算。你知道这该如何操作吗？不知道的话请再来问我。
3. 这个例子中用到了m_map。你用过m_map吗？


