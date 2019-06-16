对照论文中的示例图和文章给出的代码来梳理     
 <div align=center><img src=./pictures/PCA_1.png />   </div> 

从图中看到，整个网络有三个关键步骤，Patch-mean removal 、 PCA filter convolution与Binary quantization &mapping ，分别是局部均值化、PCA卷积、二值化与直方图映射    
im2col_mean_removal.m 
输入 
图片矩阵 InImg(m*n) 
采样矩阵大小 patchsize12=[k1,k2][k1,k2] 
图片通道数 chl(默认为1) 
在输入的矩阵InImg上，按行滑动采样矩阵，得到patch,每个patch按列展开，减去均值成为输出矩阵im的列，所以输出矩阵im为 k1∗k2k1∗k2 行，(m−k1)∗(n−k2)(m−k1)∗(n−k2) 列
```
function im = im2col_mean_removal(varargin)
% 

NumInput = length(varargin);
InImg = varargin{1};
patchsize12 = varargin{2}; 

z = size(InImg,3);
im = cell(z,1);
if NumInput == 2
    for i = 1:z
        iim = im2colstep(InImg(:,:,i),patchsize12); %窗口采样,然后按列展开为列向量
        im{i} = bsxfun(@minus, iim, mean(iim))';    %去均值
%         iim = bsxfun(@minus, iim, mean(iim)); 
%         im{i} = bsxfun(@minus, iim, mean(iim,2))';
    end
else
    for i = 1:z
        iim = im2colstep(InImg(:,:,i),patchsize12,varargin{3});
        im{i} = bsxfun(@minus, iim, mean(iim))'; 
%         iim = bsxfun(@minus, iim, mean(iim)); 
%         im{i} = bsxfun(@minus, iim, mean(iim,2))';
    end 
end
im = [im{:}]';
```

PCA_FilterBank.m 
输入为InImg （cell structure） 
PatchSize 采样窗口大小 
NumFilter 提取主成分个数 
对于每一张图片，调用前面的im2col_mean_removal.m函数，得到 
X=[X1,X2,X3…Xn]∈Rk1k2∗NmnX=[X1,X2,X3…Xn]∈Rk1k2∗Nmn 
然后对XTXXTX做主成分分解，得到特征值最大的前NumFilter个特征向量，特征向量为1∗k1k2
```
function V = PCA_FilterBank(InImg, PatchSize, NumFilters) 
% =======INPUT=============
% InImg            Input images (cell structure)  
% PatchSize        the patch size, asumed to an odd number.
% NumFilters       the number of PCA filters in the bank.
% =======OUTPUT============
% V                PCA filter banks, arranged in column-by-column manner
% =========================

addpath('./Utils')

% to efficiently cope with the large training samples, if the number of training we randomly subsample 10000 the
% training set to learn PCA filter banks
ImgZ = length(InImg);
MaxSamples = 100000;
NumRSamples = min(ImgZ, MaxSamples); 
RandIdx = randperm(ImgZ);
RandIdx = RandIdx(1:NumRSamples);

%% Learning PCA filters (V)
NumChls = size(InImg{1},3);
Rx = zeros(NumChls*PatchSize^2,NumChls*PatchSize^2); %保存均值采样后的矩阵

for i = RandIdx %1:ImgZ
    im = im2col_mean_removal(InImg{i},[PatchSize PatchSize]); % collect all the patches of the ith image in a matrix, and perform patch mean removal
    %%这里的Rx即对应文章中的 X 矩阵
    Rx = Rx + im*im'; % sum of all the input images' covariance matrix
end
Rx = Rx/(NumRSamples*size(im,2));
[E D] = eig(Rx);    %%特征值分解
[~, ind] = sort(diag(D),'descend');

%%取前NumFilters个特征向量，作为下一步的滤波器
V = E(:,ind(1:NumFilters));  % principal eigenvectors 
```

PCA_output.m 
输入InImg 图片矩阵 
InImgIdx InImg矩阵的Id号 
PatchSize 采样窗口大小 
NumFilters 前一步提取的特征向量的个数 
V 前一步提取的特征向量 
对输入的每张图片矩阵，对其边界填充0，以调整其大小为 (m+k1−1)∗(n+k2−1)(m+k1−1)∗(n+k2−1)，然后调用im2col_mean_removal.m函数，得到im矩阵(k1k2∗mn)(k1k2∗mn)，im矩阵和前一步得到的NumFilters 个特征向量分别作卷积，然后恢复为m∗nm∗n的矩阵，最后输出的卷积后图像数目为输入数目的NumFilters倍
```
function [OutImg OutImgIdx] = PCA_output(InImg, InImgIdx, PatchSize, NumFilters, V)
% Computing PCA filter outputs
% ======== INPUT ============
% InImg         Input images (cell structure); each cell can be either a matrix (Gray) or a 3D tensor (RGB)   
% InImgIdx      Image index for InImg (column vector)
% PatchSize     Patch size (or filter size); the patch is set to be sqaure
% NumFilters    Number of filters at the stage right before the output layer 
% V             PCA filter banks (cell structure); V{i} for filter bank in the ith stage  
% ======== OUTPUT ===========
% OutImg           filter output (cell structure)
% OutImgIdx        Image index for OutImg (column vector)
% OutImgIND        Indices of input patches that generate "OutImg"
% ===========================
addpath('./Utils')

ImgZ = length(InImg);
mag = (PatchSize-1)/2;
OutImg = cell(NumFilters*ImgZ,1); 
cnt = 0;
for i = 1:ImgZ
    [ImgX, ImgY, NumChls] = size(InImg{i});
    img = zeros(ImgX+PatchSize-1,ImgY+PatchSize-1, NumChls);

    %%矩阵周围填充0，保证卷积后的图像大小和原图像大小一致
    img((mag+1):end-mag,(mag+1):end-mag,:) = InImg{i};     
    im = im2col_mean_removal(img,[PatchSize PatchSize]); % collect all the patches of the ith image in a matrix, and perform patch mean removal
    for j = 1:NumFilters
     %% 和每一个滤波器做卷积
        OutImg{cnt} = reshape(V(:,j)'*im,ImgX,ImgY);  % convolution output
    end
    InImg{i} = [];   %%释放内存空间
end
OutImgIdx = kron(InImgIdx,ones(NumFilters,1));   %%记录每张图片的id
```

HashingHist.m 
输入OutImg 图像矩阵 
ImgIdx id号 
对最后得到的NumFilters(1)∗NumFilters(2)NumFilters(1)∗NumFilters(2)个特征map矩阵，先用Heaviside.m这个函数做二值化(大于0映射为1，否则为0)，然后每个矩阵乘以一个权值后相加，得到矩阵T,再对T做HistBlockSize窗口大小的采样，得到新矩阵(类似函数im2col_mean_removal.m，只是少了去均值的步骤)，最后调用histc，做直方图，归一化得到最后的特征向量
```
function [f BlkIdx] = HashingHist(PCANet,ImgIdx,OutImg)
% Output layer of PCANet (Hashing plus local histogram)
% ========= INPUT ============
% PCANet  PCANet parameters (struct)
%       .PCANet.NumStages      
%           the number of stages in PCANet; e.g., 2  
%       .PatchSize
%           the patch size (filter size) for square patches; e.g., [5 3]
%           means patch size equalt to 5 and 3 in the first stage and second stage, respectively 
%       .NumFilters
%           the number of filters in each stage; e.g., [16 8] means 16 and
%           8 filters in the first stage and second stage, respectively
%       .HistBlockSize 
%           the size of each block for local histogram; e.g., [10 10]
%       .BlkOverLapRatio 
%           overlapped block region ratio; e.g., 0 means no overlapped 
%           between blocks, and 0.3 means 30% of blocksize is overlapped 
%       .Pyramid
%           spatial pyramid matching; e.g., [1 2 4], and [] if no Pyramid
%           is applied
% ImgIdx  Image index for OutImg (column vector)
% OutImg  PCA filter output before the last stage (cell structure)
% ========= OUTPUT ===========
% f       PCANet features (each column corresponds to feature of each image)
% BlkIdx  index of local block from which the histogram is compuated
% ============================
addpath('./Utils')


NumImg = max(ImgIdx);
f = cell(NumImg,1);
map_weights = 2.^((PCANet.NumFilters(end)-1):-1:0); % weights for binary to decimal conversion

for Idx = 1:NumImg

    Idx_span = find(ImgIdx == Idx);
    NumOs = length(Idx_span)/PCANet.NumFilters(end); % the number of "O"s
    Bhist = cell(NumOs,1);

    for i = 1:NumOs 

        T = 0;
        ImgSize = size(OutImg{Idx_span(PCANet.NumFilters(end)*(i-1) + 1)});
        for j = 1:PCANet.NumFilters(end)
            T = T + map_weights(j)*Heaviside(OutImg{Idx_span(PCANet.NumFilters(end)*(i-1)+j)}); 
            % weighted combination; hashing codes to decimal number conversion

            OutImg{Idx_span(PCANet.NumFilters(end)*(i-1)+j)} = [];
        end

        if isempty(PCANet.HistBlockSize)
            NumBlk = ceil((PCANet.ImgBlkRatio - 1)./PCANet.BlkOverLapRatio) + 1;
            HistBlockSize = ceil(size(T)./PCANet.ImgBlkRatio);
            OverLapinPixel = ceil((size(T) - HistBlockSize)./(NumBlk - 1));
            NImgSize = (NumBlk-1).*OverLapinPixel + HistBlockSize;
            Tmp = zeros(NImgSize);
            Tmp(1:size(T,1), 1:size(T,2)) = T;
            Bhist{i} = sparse(histc(im2col_general(Tmp,HistBlockSize,...
            OverLapinPixel),(0:2^PCANet.NumFilters(end)-1)')); 
        else 

            stride = round((1-PCANet.BlkOverLapRatio)*PCANet.HistBlockSize); 
            %%  得到直方图
            blkwise_fea = sparse(histc(im2col_general(T,PCANet.HistBlockSize,...
              stride),(0:2^PCANet.NumFilters(end)-1)')); 
            % calculate histogram for each local block in "T"

           if ~isempty(PCANet.Pyramid)
                x_start = ceil(PCANet.HistBlockSize(2)/2);
                y_start = ceil(PCANet.HistBlockSize(1)/2);
                x_end = floor(ImgSize(2) - PCANet.HistBlockSize(2)/2);
                y_end = floor(ImgSize(1) - PCANet.HistBlockSize(1)/2);

                sam_coordinate = [...
                    kron(x_start:stride:x_end,ones(1,length(y_start:stride: y_end))); 
                    kron(ones(1,length(x_start:stride:x_end)),y_start:stride: y_end)];               

                blkwise_fea = spp(blkwise_fea, sam_coordinate, ImgSize, PCANet.Pyramid)';

           else
                blkwise_fea = bsxfun(@times, blkwise_fea, ...
                    2^PCANet.NumFilters(end)./sum(blkwise_fea)); 
           end

           Bhist{i} = blkwise_fea;
        end

    end           
    f{Idx} = vec([Bhist{:}]');

    if ~isempty(PCANet.Pyramid)
        f{Idx} = sparse(f{Idx}/norm(f{Idx}));
    end
end
f = [f{:}];

if ~isempty(PCANet.Pyramid)
    BlkIdx = kron((1:size(Bhist{1},1))',ones(length(Bhist)*size(Bhist{1},2),1));
else
    BlkIdx = kron(ones(NumOs,1),kron((1:size(Bhist{1},2))',ones(size(Bhist{1},1),1)));
end

%-------------------------------
function X = Heaviside(X) % binary quantization
X = sign(X);
X(X<=0) = 0;

function x = vec(X) % vectorization
x = X(:);


function beta = spp(blkwise_fea, sam_coordinate, ImgSize, pyramid)

[dSize, ~] = size(blkwise_fea);

img_width = ImgSize(2);
img_height = ImgSize(1);

% spatial levels
pyramid_Levels = length(pyramid);
pyramid_Bins = pyramid.^2;
tBins = sum(pyramid_Bins);

beta = zeros(dSize, tBins);
cnt = 0;

for i1 = 1:pyramid_Levels,

    Num_Bins = pyramid_Bins(i1);

    wUnit = img_width / pyramid(i1);
    hUnit = img_height / pyramid(i1);

    % find to which spatial bin each local descriptor belongs
    xBin = ceil(sam_coordinate(1,:) / wUnit);
    yBin = ceil(sam_coordinate(2,:) / hUnit);
    idxBin = (yBin - 1)*pyramid(i1) + xBin;

    for i2 = 1:Num_Bins,     
        cnt = cnt + 1;
        sidxBin = find(idxBin == i2);
        if isempty(sidxBin),
            continue;
        end      
        beta(:, cnt) = max(blkwise_fea(:, sidxBin), [], 2);
    end
end
```
PCANet_train.m 
训练函数，分两个stage,第一个stage,调用PCA_FilterBank，PCA_output，合起来就是一个PCA滤波；第二个stage,和第一步相同，滤波后再用HashingHist，做直方图，得到最终的特征向量    
```
function [f V BlkIdx] = PCANet_train(InImg,PCANet,IdtExt)
% =======INPUT=============
% InImg     Input images (cell); each cell can be either a matrix (Gray) or a 3D tensor (RGB)  
% PCANet    PCANet parameters (struct)
%       .PCANet.NumStages      
%           the number of stages in PCANet; e.g., 2  
%       .PatchSize
%           the patch size (filter size) for square patches; e.g., [5 3]
%           means patch size equalt to 5 and 3 in the first stage and second stage, respectively 
%       .NumFilters
%           the number of filters in each stage; e.g., [16 8] means 16 and
%           8 filters in the first stage and second stage, respectively
%       .HistBlockSize 
%           the size of each block for local histogram; e.g., [10 10]
%       .BlkOverLapRatio 
%           overlapped block region ratio; e.g., 0 means no overlapped 
%           between blocks, and 0.3 means 30% of blocksize is overlapped 
%       .Pyramid
%           spatial pyramid matching; e.g., [1 2 4], and [] if no Pyramid
%           is applied
% IdtExt    a number in {0,1}; 1 do feature extraction, and 0 otherwise  
% =======OUTPUT============
% f         PCANet features (each column corresponds to feature of each image)
% V         learned PCA filter banks (cell)
% BlkIdx    index of local block from which the histogram is compuated
% =========================

addpath('./Utils')


if length(PCANet.NumFilters)~= PCANet.NumStages;
    display('Length(PCANet.NumFilters)~=PCANet.NumStages')
    return
end

NumImg = length(InImg);  %总的训练图片的数目

V = cell(PCANet.NumStages,1); 
OutImg = InImg; 
ImgIdx = (1:NumImg)';
clear InImg; 

for stage = 1:PCANet.NumStages
    display(['Computing PCA filter bank and its outputs at stage ' num2str(stage) '...'])

    V{stage} = PCA_FilterBank(OutImg, PCANet.PatchSize(stage), PCANet.NumFilters(stage)); % compute PCA filter banks

    if stage ~= PCANet.NumStages % compute the PCA outputs only when it is NOT the last stage
        [OutImg ImgIdx] = PCA_output(OutImg, ImgIdx, ...
            PCANet.PatchSize(stage), PCANet.NumFilters(stage), V{stage});  
    end
end

if IdtExt == 1 % enable feature extraction
    %display('PCANet training feature extraction...') 

    f = cell(NumImg,1); % compute the PCANet training feature one by one 

    for idx = 1:NumImg
        if 0==mod(idx,100); display(['Extracting PCANet feasture of the ' num2str(idx) 'th training sample...']); end
        OutImgIndex = ImgIdx==idx; % select feature maps corresponding to image "idx" (outputs of the-last-but-one PCA filter bank) 

        [OutImg_i ImgIdx_i] = PCA_output(OutImg(OutImgIndex), ones(sum(OutImgIndex),1),...
            PCANet.PatchSize(end), PCANet.NumFilters(end), V{end});  % compute the last PCA outputs of image "idx"

        [f{idx} BlkIdx] = HashingHist(PCANet,ImgIdx_i,OutImg_i); % compute the feature of image "idx"
         fprintf('%d %d\n',size(f{idx}));%print(size(f{idx}));
%        [f{idx} BlkIdx] = SphereSum(PCANet,ImgIdx_i,OutImg_i); % Testing!!
        OutImg(OutImgIndex) = cell(sum(OutImgIndex),1); 

    end
    f = sparse([f{:}]);

else  % disable feature extraction
    f = [];
    BlkIdx = [];
end
```

http://my.oschina.net/Ldpe2G/blog/275922?p=4#comments     
http://blog.csdn.net/u013088062/article/details/50039573     
http://blog.csdn.net/zxdxyz/article/details/40479343     
http://my.oschina.net/Ldpe2G/blog/275922    

