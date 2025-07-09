본 리포지토리는 [JamesnetGroup](https://github.com/JamesnetGroup)의 [navigationbar](https://github.com/JamesnetGroup/navigationbar)를 개조해서 복합 배경색 및 반투명 메뉴 영역을 적용할 수 있도록 만든 예제입니다.

---
작업 중인 어떤 프로젝트에서 메뉴를 좀 예쁘게 만들어보려던 찰나에, [vickyqu115](https://github.com/vickyqu115)님께서 공유해 주신 Magic NavigationBar가 떠올라 [jamesnet214](https://github.com/jamesnet214)님의 유튜브 채널 영상과 함께 참고하기로 했었습니다.

그런데 제가 평소에 배경을 단색으로 안 쓰고 이미지나 그라데이션, 패턴 등을 사용하는 습관이 좀 있습니다. 그리고 그 위에 컨텐츠의 패널을 조금 반투명하게 표현하는 것도 종종 사용하죠. 그래서 배경을 패턴으로 설정하고, 메뉴 영역을 조금 반투명하게 하면 다음의 그림과 같이 나오게 됩니다.

![패턴배경적용](https://github.com/user-attachments/assets/f4de9b65-35fe-4f6e-963e-fef5cd71a1da)


기존의 방식은 선택된 메뉴 영역을 배경 색상과 똑같은 도형으로 오버랩 하는 방식이라서, 저의 개인 스타일을 고수할 수 없었습니다. ㅎㅎ

그래서 [CombinedGeometry](https://learn.microsoft.com/ko-kr/dotnet/api/system.windows.media.combinedgeometry)와 [Transform](https://learn.microsoft.com/ko-kr/dotnet/desktop/wpf/graphics-multimedia/transforms-overview)들의 조합을 이용해서 약간 다른 방식을 적용해봤습니다. 일단 구현된 결과는 다음과 같습니다.

![개선결과](https://github.com/user-attachments/assets/e05e0490-d5d4-4d81-b62f-9738244f8a2b)


기본 원리는, 우선 아래의 그림과 같이 파란색으로 표시한 메뉴 영역의 복사본을 두 개 만듭니다. 그리고 각각 (X: -10, Y: -10), (X: 10, Y: 10) 만큼 이동시킨 후 GeometryCombineMode를 Intersect로 설정해서 교집합 연산을 수행합니다.

![교집합](https://github.com/user-attachments/assets/c4e67a67-a61c-484f-9f8b-94b7e662c32a)


그리고 교집합 연산 결과에서, 적절한 위치로 변환된 EllipseGeometry 영역을 GeometryCombineMode.Exclude를 이용해 차집합 연산을 합니다. EllipseGeometry 영역의 위치 변환은 ListBox의 SelectedIndex 값을 해당 영역의 크기로 확대 변환하여 적용하였습니다. 결과는 다음과 같습니다.

![차집합](https://github.com/user-attachments/assets/aa8486f4-c419-4a88-a093-02a23f586eef)


그리고 차집합 연산 결과로 생성된 Path 객체의 테두리를 두껍게 만들어줍니다.

![테두리](https://github.com/user-attachments/assets/7a10c0f9-ae8e-40ae-96b8-9fc7693ba7fc)

이렇게 하면 꼭짓점을 둥글게 깎은 것과 같은 효과를 얻을 수 있습니다. 또한 직선과 원호의 교점도 자연스럽게 효과가 적용됩니다.

이렇게 만들어진 Path의 Fill과 Stroke에 동일한 브러시를 바인딩하면 선택 영역이 동그랗게 깎인 최종 결과물을 얻을 수 있습니다. 다만, Fill과 Stroke에 사용된 색상이 반투명할 경우 면과 선이 중첩되어 원하는 결과물을 얻을 수 없을 수도 있습니다. 다음의 그림처럼 말이죠.

![테두리오류](https://github.com/user-attachments/assets/0b3155d5-bea5-439d-980d-212634be28e0)


따라서 해당 Path를 적용한 VisualBrush를 OpacityMask로 사용하기로 했습니다.

```xaml
<Rectangle Fill="{TemplateBinding Background}" Height="80" VerticalAlignment="Bottom">
    <Rectangle.OpacityMask>
        <VisualBrush>
            <VisualBrush.Visual>
                <Path Fill="White" Stroke="White" StrokeThickness="20" StrokeLineJoin="Round">
                    <Path.Data>
                        <CombinedGeometry GeometryCombineMode="Exclude">
                            <CombinedGeometry.Geometry1>
                                <CombinedGeometry GeometryCombineMode="Intersect">
                                    <CombinedGeometry.Geometry1>
                                        <RectangleGeometry x:Name="BarBorder" Rect="{Binding RenderedGeometry.Bounds, Mode=OneTime, ElementName=Bar}">
                                            <RectangleGeometry.Transform>
                                                <TranslateTransform X="-10" Y="-10"/>
                                            </RectangleGeometry.Transform>
                                        </RectangleGeometry>
                                    </CombinedGeometry.Geometry1>
                                    <CombinedGeometry.Geometry2>
                                        <RectangleGeometry Rect="{Binding Rect, ElementName=BarBorder}">
                                            <RectangleGeometry.Transform>
                                                <TranslateTransform X="10" Y="10"/>
                                            </RectangleGeometry.Transform>
                                        </RectangleGeometry>
                                    </CombinedGeometry.Geometry2>
                                </CombinedGeometry>
                            </CombinedGeometry.Geometry1>
                            <CombinedGeometry.Geometry2>
                                <EllipseGeometry RadiusX="0.635" RadiusY="0.635">
                                    <EllipseGeometry.Transform>
                                        <TransformGroup>
                                            <TranslateTransform
                                                x:Name="Translate"
                                                X="{Binding SelectedIndex, Mode=OneTime, RelativeSource={RelativeSource Mode=TemplatedParent}}"/>
                                            <ScaleTransform ScaleX="80" ScaleY="80"/>
                                            <TranslateTransform X="60"/>
                                        </TransformGroup>
                                    </EllipseGeometry.Transform>
                                </EllipseGeometry>
                            </CombinedGeometry.Geometry2>
                        </CombinedGeometry>
                    </Path.Data>
                </Path>
            </VisualBrush.Visual>
        </VisualBrush>
    </Rectangle.OpacityMask>
</Rectangle>
```

참고로 커스텀 컨트롤로 따로 안 만들고 그냥 ListBox 스타일로 구현해서 C# 코드 필요 없이 그냥 리소스 사전 내용만 복사해서 쓸 수 있게 만들었습니다.

```xaml
<Window.Resources>
    <ResourceDictionary>
        <ResourceDictionary.MergedDictionaries>
            <ResourceDictionary Source="/Themes/Icons.xaml"/>
            <ResourceDictionary Source="/Themes/NavigationBar.xaml"/>
        </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
</Window.Resources>

<StackPanel VerticalAlignment="Center">

    <ListBox Margin="20" SelectedIndex="2" Background="#AADDDDDD">
        <ListBoxItem Content="Microsoft" Tag="{StaticResource Icons.Microsoft}"/>
        <ListBoxItem Content="Apple" Tag="{StaticResource Icons.Apple}"/>
        <ListBoxItem Content="Google" Tag="{StaticResource Icons.Google}"/>
        <ListBoxItem Content="Facebook" Tag="{StaticResource Icons.Facebook}"/>
        <ListBoxItem Content="Instagram" Tag="{StaticResource Icons.Instagram}"/>
    </ListBox>

    <ListBox Margin="20" SelectedIndex="1" Background="#AA222222" Foreground="IndianRed">
        <ListBox.Resources>
            <Style TargetType="{x:Type ListBoxItem}" BasedOn="{StaticResource {x:Type ListBoxItem}}">
                <Setter Property="Foreground" Value="White"/>
            </Style>
        </ListBox.Resources>
        <ListBoxItem Content="Microsoft" Tag="{StaticResource Icons.Microsoft}"/>
        <ListBoxItem Content="Apple" Tag="{StaticResource Icons.Apple}"/>
        <ListBoxItem Content="Google" Tag="{StaticResource Icons.Google}"/>
    </ListBox>
        
</StackPanel>
```

그리고 다른 종속 패키지 없이 바로 복사해서 쓸 수 있게 하려고 했습니다만, 안타깝게도 [Microsoft.Xaml.Behaviors.Wpf](https://www.nuget.org/packages/Microsoft.Xaml.Behaviors.Wpf/) 패키지를 쓸 수밖에 없었습니다.

컨트롤 템플릿 트리거의 애니메이션에 바인딩을 적용하면 ‘스레드에서 사용하기 위해 이 Storyboard 시간 표시 막대 트리를 고정할 수 없습니다.’ 오류가 발생하기 때문에 어쩔 수 없이 컨트롤 템플릿 트리거 대신 [Microsoft.Xaml.Behaviors.Wpf](https://www.nuget.org/packages/Microsoft.Xaml.Behaviors.Wpf/) 패키지에 구현된 Interaction.Triggers를 이용했습니다.

```xaml
...
<ControlTemplate TargetType="{x:Type ListBox}">
    <Grid Height="120" UseLayoutRounding="False" SnapsToDevicePixels="False">
        <i:Interaction.Triggers>
            <i:PropertyChangedTrigger Binding="{Binding SelectedItem, RelativeSource={RelativeSource Mode=TemplatedParent}}">
                <i:ControlStoryboardAction>
                    <i:ControlStoryboardAction.Storyboard>
                        <Storyboard>
                            <DoubleAnimation Storyboard.TargetName="Translate" Storyboard.TargetProperty="X"
                                                To="{Binding SelectedIndex, RelativeSource={RelativeSource Mode=TemplatedParent}}" 
                                                Duration="0:0:0.5">
                                <DoubleAnimation.EasingFunction>
                                    <QuinticEase EasingMode="EaseInOut"/>
                                </DoubleAnimation.EasingFunction>
                            </DoubleAnimation>
                        </Storyboard>
                    </i:ControlStoryboardAction.Storyboard>
                </i:ControlStoryboardAction>
            </i:PropertyChangedTrigger>
        </i:Interaction.Triggers>
...
</ControlTemplate>
...
```

리포지토리 이름은 원본 소스코드 출처인 [JamesnetGroup](https://github.com/JamesnetGroup)의 이름을 따서 JamesnetNavigationBar로 명명했습니다.

[CombinedGeometry](https://learn.microsoft.com/ko-kr/dotnet/api/system.windows.media.combinedgeometry)와 [Transform](https://learn.microsoft.com/ko-kr/dotnet/desktop/wpf/graphics-multimedia/transforms-overview) 조합 등을 다른 프로젝트에 응용하실 때 참고자료로서 도움이 되었으면 좋겠습니다.
