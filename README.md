graph TD
    %% Inputs
    VI[可见光图像 vi] --> VisStage1
    IR[红外图像 ir] --> IrStage1

    %% ------------------- Encoder -------------------
    subgraph Encoder [双分支特征提取]
        subgraph Stage1
            VisStage1[ACmix 卷积] --> CBAM_V1[CBAM 注意力]
            IrStage1[ACmix 卷积] --> CBAM_I1[CBAM 注意力]
            CBAM_V1 --> Skip1((Skip 1 <br> Concat))
            CBAM_I1 --> Skip1
        end

        subgraph Stage2
            CBAM_V1 --> Down1_V[DownSample] --> RDAB_V[RDAB 模块] --> CBAM_V2[CBAM 注意力]
            CBAM_I1 --> Down1_I[DownSample] --> RDAB_I[RDAB 模块] --> CBAM_I2[CBAM 注意力]
            CBAM_V2 --> Skip2((Skip 2 <br> Concat))
            CBAM_I2 --> Skip2
        end

        subgraph Stage3
            CBAM_V2 --> Down2_V[DownSample] --> ConvL_V[ConvLeakyRelu] --> Vis3[vis3 深度特征]
            CBAM_I2 --> Down2_I[DownSample] --> ConvL_I[ConvLeakyRelu] --> Ir3[ir3 深度特征]
        end
    end

    %% ------------------- Latent Fusion -------------------
    subgraph Bottleneck [深度特征融合层]
        Vis3 --> ADFM[ADFM <br> 自适应差分融合]
        Ir3 --> ADFM
        
        Vis3 --> SwinCross[SwinCrossDomainFusion <br> 跨域 Swin Transformer]
        Ir3 --> SwinCross
        
        ADFM --> F_MID((f_mid <br> Concat))
        SwinCross --> F_MID
    end

    %% ------------------- Decoder -------------------
    subgraph Decoder [竞争式专家解码器 CompetitiveExpertDecoder]
        F_MID --> Canvas[Canvas Init <br> 特征初始化]
        Skip1 --> Canvas
        Skip2 --> Canvas
        
        Canvas --> Exp_Struct[Structure Expert <br> 结构专家]
        Canvas --> Exp_Therm[Thermal Target Expert <br> 热目标专家]
        Canvas --> Exp_Text[Texture Natural Expert <br> 纹理专家]
        
        %% 专家额外输入
        SwinCross -.引导.-> Exp_Therm
        ADFM -.引导.-> Exp_Therm
        IR -.原图引导.-> Exp_Therm
        VI -.原图引导.-> Exp_Text
        
        Exp_Struct --> Comp[Expert Competition Fusion <br> 专家竞争融合]
        Exp_Therm --> Comp
        Exp_Text --> Comp
        
        Comp --> Final[Fused Image <br> 最终融合图像]
    end
