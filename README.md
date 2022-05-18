# Qt-OpenGL-

t新渲染底层Scene Graph研究（三）
上一篇文章介绍了Qt Quick和SceneGraph的一些理论上的内容。这也是我最新的研究成果。接下来我要介绍一下如何使用Scene Graph来制作一些好玩的效果。这也是我进行一次SceneGraph的尝试。

我的目标是希望在Scene Graph这一套渲染框架下实现一个带有纹理的立方体，并且旋转。花了几天，虽然不是那么满意，但是已经告一段落了。

蒋彩阳原创文章，首发地址：http://blog.csdn.net/gamesdev/article/details/43196787。欢迎同行前来探讨。

本文难度偏大，适合有经验的同行进行交流。

首先，对比C++，QML这边的代码稍微简单一些，那么从最简单开始说起吧。

import QtQuick 2.4
import QtQuick.Window 2.2
import TexturedCube 1.0

Window
{
    title: qsTr( "Scene Graph Textured Object" )
    width: isMobileDevice( )? Screen.width: 480
    height: isMobileDevice( )? Screen.height: 320
    visible: true

    Rectangle
    {
        anchors.fill: parent
        color: "orange"

        Cube
        {
            id: theCube
            anchors.centerIn: parent
            length: 60
            source: "../image/avatar.jpg"

            property int rotateAngle: 0
            transform:
                [
                Rotation
                {
                    angle: theCube.rotateAngle
                    axis
                    {
                        x: 1
                        y: 1
                    }
                }
            ]

            NumberAnimation on rotateAngle
            {
                from: 0
                to: 360
                duration: 1000
                loops: Animation.Infinite
            }

            Timer
            {
                interval: 3000
                repeat: true
                property int tCount: 0
                running: true
                onTriggered:
                {
                    var sourceList = [ "../image/soul.png",
                             "../image/avatar.jpg" ];
                    theCube.source = sourceList[tCount++ % 2];
                }
            }
        }
    }

    function isMobileDevice( )// 判断是否是移动平台
    {
        return  Qt.platform.os === "android" ||
                Qt.platform.os === "blackberry" ||
                Qt.platform.os === "ios" ||
                Qt.platform.os === "winphone";
    }
}
一个普通的窗口，背景是橙色的，在上面显示了我们的Cube。我希望我的Cube沿着一个轴进行旋转，所以设定了NumberAnimationon rotateAngle。此外，我希望每隔三秒Cube更换纹理，所以设定了一个Timer来更换纹理。每一个Item都有transform成员，它表示Item经过什么样的转换，目前transform支持Translation、Rotation以及Scale，有人想要让MouseArea成为不规则的，其实如果官方提供了Shear这个类，那么就更方便了。

我们看到的QML代码仅仅是表象，其实在幕后，是一个较为复杂的C++类：TexturedCube。下面我们再来看看TexturedCube.h的内容：

#ifndef TEXTUREDCUBE
#define TEXTUREDCUBE

#include <QUrl>
#include <QQuickItem>

#define DECLRARE_QUICKITEM_PROPERTY( aType, aProperty ) protected:
    aType m_ ## aProperty; public: 
    aType aProperty( void ) { return m_ ## aProperty; } 
    void set ## aProperty( aType _ ## aProperty ) 
    {
        if ( m_ ## aProperty == _ ## aProperty ) return; 
        m_ ## aProperty = _ ## aProperty; 
        emit aProperty ## Changed( _ ## aProperty ); 
        update( ); 
    }

class TexturedCube: public QQuickItem
{
    Q_OBJECT
    Q_PROPERTY( qreal length READ Length
                WRITE setLength NOTIFY LengthChanged )
    Q_PROPERTY( QUrl source READ Source
                WRITE setSource NOTIFY SourceChanged )
public:
    explicit TexturedCube( void );

    QSGNode* updatePaintNode( QSGNode* oldNode,
                              UpdatePaintNodeData* );
private slots:
    void checkTextureReconstruct( QUrl source );
signals:
    void LengthChanged( qreal length );
    void SourceChanged( QUrl source );
private:
    QString source2Path( QUrl source );

    QUrl        m_LastSource;
    bool        m_TextureNeedsReconstruct;
protected:
    DECLRARE_QUICKITEM_PROPERTY( qreal, Length )
    DECLRARE_QUICKITEM_PROPERTY( QUrl, Source )
};

#endif // TEXTUREDCUBE

在头文件我首先定义了一个方便的宏，包含了属性：Getter-Setter，以及Notifier。和QQuickItem的大多数属性一样，一旦某个属性发生了改变，那么就要发出改变的信号，也就是我们称的notifier，并且要告诉QQuickItem进行下一次更新。这里的update()并不是立即更新的意思，而是告诉QQuickItem在下个循环周期之前调用updatePaintNode()这个函数。接下来我们看看TexturedCube.cpp文件。

// TexturedCube.cpp
#include <QQmlFile>
#include <QSGGeometryNode>
#include <QSGGeometry>
#include <QSGOpaqueTextureMaterial>
#include <QQuickWindow>
#include "TexturedCube.h"

#define VERTEX_COUNT        36

TexturedCube::TexturedCube( void )
{
    m_Length = 10.0;
    m_Source = "";
    setFlag( ItemHasContents, true );
    connect( this, SIGNAL( SourceChanged( QUrl ) ),
             this, SLOT( checkTextureReconstruct( QUrl ) ) );
}

struct TexturedCubeVertex
{
    QVector3D               position;
    QVector2D               texCoord;
};

QSGNode *TexturedCube::updatePaintNode( QSGNode* oldNode,
                                        QQuickItem::UpdatePaintNodeData* )
{
    QSGGeometryNode* node = Q_NULLPTR;
    QSGGeometry* geometry = Q_NULLPTR;

    if ( oldNode == Q_NULLPTR )
    {
        // 创建几何体
        static QSGGeometry::Attribute attributes[] =
        {
            QSGGeometry::Attribute::create(
            0,                  // 数组元素的下标
            3,                  // 元的个数
            GL_FLOAT,           // 元的类型
            true ),             // 是不是顶点位置元素
            QSGGeometry::Attribute::create(
            1,                  // 数组元素的下标
            2,                  // 元的个数
            GL_FLOAT,           // 元的类型
            false )             // 是不是顶点位置元素
        };
        static QSGGeometry::AttributeSet attributeSet =
        {
            2,                              // 属性的个数
            sizeof( TexturedCubeVertex ),   // 个性化顶点结构的大小
            attributes                      // 属性的数组
        };
        geometry = new QSGGeometry(
                    attributeSet,       // 属性集
                    VERTEX_COUNT );     // 顶点个数
        geometry->setDrawingMode( GL_TRIANGLES );
        geometry->setVertexDataPattern( QSGGeometry::DynamicPattern );
        geometry->allocate( VERTEX_COUNT );

        // 创建材质
        QImage image( source2Path( m_Source ) );
        QSGTexture* texture = window( )->createTextureFromImage(
                    image.mirrored( ) );
        texture->setParent( window( ) );
        QSGOpaqueTextureMaterial* material = new QSGOpaqueTextureMaterial;
        material->setTexture( texture );

        // 创建节点
        node = new QSGGeometryNode;
        node->setGeometry( geometry );
        node->setMaterial( material );
        node->setFlag( QSGNode::OwnsGeometry );
        node->setFlag( QSGNode::OwnsOpaqueMaterial );
        node->markDirty( QSGNode::DirtyGeometry | QSGNode::DirtyMaterial );
    }
    else
    {
        node = static_cast<QSGGeometryNode*>( oldNode );
        geometry = node->geometry( );
        QSGNode::DirtyState dirtyState = QSGNode::DirtyGeometry;

        if ( m_TextureNeedsReconstruct )
        {
            QSGOpaqueTextureMaterial* material =
                    static_cast<QSGOpaqueTextureMaterial*>( node->material( ) );
            if ( material != Q_NULLPTR )
            {
                QSGTexture* texture = material->texture( );
                texture->setParent( Q_NULLPTR );
                delete texture;
                QImage image( source2Path( m_Source ) );
                texture = window( )->createTextureFromImage(
                                    image.mirrored( ) );
                texture->setParent( window( ) );
                material->setTexture( texture );
                dirtyState |= QSGNode::DirtyMaterial;
            }
        }

        node->markDirty( dirtyState );
    }

    TexturedCubeVertex* v = static_cast<TexturedCubeVertex*>( geometry->vertexData( ) );

    // 设置顶点坐标
    qreal semi = m_Length / 2.0;
    const QVector3D basicVertices[] =
    {
        QVector3D( semi, -semi, semi ),
        QVector3D( semi, -semi, -semi ),
        QVector3D( -semi, -semi, -semi ),
        QVector3D( -semi, -semi, semi ),
        QVector3D( semi, semi, semi ),
        QVector3D( semi, semi, -semi ),
        QVector3D( -semi, semi, -semi ),
        QVector3D( -semi, semi, semi )
    };

    // 前面
    v[0].position = basicVertices[6]; v[0].texCoord = QVector2D( 0.0, 0.0 );
    v[1].position = basicVertices[2]; v[1].texCoord = QVector2D( 0.0, 1.0 );
    v[2].position = basicVertices[5]; v[2].texCoord = QVector2D( 1.0, 0.0 );
    v[3].position = basicVertices[2]; v[3].texCoord = QVector2D( 0.0, 1.0 );
    v[4].position = basicVertices[1]; v[4].texCoord = QVector2D( 1.0, 1.0 );
    v[5].position = basicVertices[5]; v[5].texCoord = QVector2D( 1.0, 0.0 );

    // 后面
    v[6].position = basicVertices[4]; v[6].texCoord = QVector2D( 0.0, 0.0 );
    v[7].position = basicVertices[0]; v[7].texCoord = QVector2D( 0.0, 1.0 );
    v[8].position = basicVertices[7]; v[8].texCoord = QVector2D( 1.0, 0.0 );
    v[9].position = basicVertices[0]; v[9].texCoord = QVector2D( 0.0, 1.0 );
    v[10].position = basicVertices[3]; v[10].texCoord = QVector2D( 1.0, 1.0 );
    v[11].position = basicVertices[7]; v[11].texCoord = QVector2D( 1.0, 0.0 );

    // 上面
    v[12].position = basicVertices[2]; v[12].texCoord = QVector2D( 0.0, 0.0 );
    v[13].position = basicVertices[3]; v[13].texCoord = QVector2D( 0.0, 1.0 );
    v[14].position = basicVertices[1]; v[14].texCoord = QVector2D( 1.0, 0.0 );
    v[15].position = basicVertices[3]; v[15].texCoord = QVector2D( 0.0, 1.0 );
    v[16].position = basicVertices[0]; v[16].texCoord = QVector2D( 1.0, 1.0 );
    v[17].position = basicVertices[1]; v[17].texCoord = QVector2D( 1.0, 0.0 );

    // 下面
    v[18].position = basicVertices[7]; v[18].texCoord = QVector2D( 0.0, 0.0 );
    v[19].position = basicVertices[6]; v[19].texCoord = QVector2D( 0.0, 1.0 );
    v[20].position = basicVertices[4]; v[20].texCoord = QVector2D( 1.0, 0.0 );
    v[21].position = basicVertices[6]; v[21].texCoord = QVector2D( 0.0, 1.0 );
    v[22].position = basicVertices[5]; v[22].texCoord = QVector2D( 1.0, 1.0 );
    v[23].position = basicVertices[4]; v[23].texCoord = QVector2D( 1.0, 0.0 );

    // 左面
    v[24].position = basicVertices[7]; v[24].texCoord = QVector2D( 0.0, 0.0 );
    v[25].position = basicVertices[3]; v[25].texCoord = QVector2D( 0.0, 1.0 );
    v[26].position = basicVertices[6]; v[26].texCoord = QVector2D( 1.0, 0.0 );
    v[27].position = basicVertices[3]; v[27].texCoord = QVector2D( 0.0, 1.0 );
    v[28].position = basicVertices[2]; v[28].texCoord = QVector2D( 1.0, 1.0 );
    v[29].position = basicVertices[6]; v[29].texCoord = QVector2D( 1.0, 0.0 );

    // 右面
    v[30].position = basicVertices[5]; v[30].texCoord = QVector2D( 0.0, 0.0 );
    v[31].position = basicVertices[1]; v[31].texCoord = QVector2D( 0.0, 1.0 );
    v[32].position = basicVertices[4]; v[32].texCoord = QVector2D( 1.0, 0.0 );
    v[33].position = basicVertices[1]; v[33].texCoord = QVector2D( 0.0, 1.0 );
    v[34].position = basicVertices[0]; v[34].texCoord = QVector2D( 1.0, 1.0 );
    v[35].position = basicVertices[4]; v[35].texCoord = QVector2D( 1.0, 0.0 );

    const QRectF bounding = boundingRect( );
    const float factor = m_Length;
    for ( int i = 0; i < VERTEX_COUNT; ++i )// 调整位置
    {
        // 这里由于坐标系x向右增大，y向下增大，符合屏幕坐标系，那么根据OpenGL右手坐标系，
        // Z轴朝里面增大。
        float x = v[i].position.x( ) + bounding.width( ) / 2;
        float y = v[i].position.y( ) + bounding.height( ) / 2;
        float z = ( v[i].position.z( ) + semi ) / factor;
        v[i].position.setX( x );
        v[i].position.setY( y );
        v[i].position.setZ( z );
    }

    return node;
}

void TexturedCube::checkTextureReconstruct( QUrl source )
{
    m_TextureNeedsReconstruct = m_LastSource != source;
}

QString TexturedCube::source2Path( QUrl source )
{
    QUrl url( source );
    if ( url.isRelative( ) )
    {
        QUrl baseURL( "file:///" + qApp->applicationFilePath( ) );
        url = baseURL.resolved( url );
    }
    return QQmlFile::urlToLocalFileOrQrc( url );
}

经过实践，要渲染一个带六个纹理面的立方体，需要36个顶点。所以我设定了一个宏VERTEX_COUNT，值是36。此外，QtScene Graph默认的渲染是AoS（Array of Structure），这一点和Direct3D9一样的，所以我们需要定义自己的顶点格式。structTexturedCubeVertex里面包含了顶点位置以及纹理坐标（又称uv坐标）。在TexturedCube的构造函数，我们通过设定setFlag(ItemHasContents, true );来告诉Scene Graph，此项目有内容，需要调用updatePaintNode()函数，然后当source改变的时候，为了调用相应的处理函数，需要连接处理器。这里的核心就是updatePaintNode()函数。这个函数的作用类似于一个中转站，火车从远处来，进站，如果有什么变化，比如说上下客，那么车内的人员也会变化。最后火车驶离车站，开往下一个目的地。此函数颇有这一个意思。第一次，我们发现oldNode指针为空，那么我们创建几何体、材质和纹理。否则oldNode指针不为空，我们认为这不是第一次渲染了，我们取出它的几何体和纹理。如果纹理路径发生了改变，那么我们也需要进行重新载入，通过source2Path()来解析路径。这里可能没有Qt源码那么复杂，因为Qt里面如果来自网络，即以http协议开始的，那么需要调用QQmlEngine里面的QNetworkAccessManager，此外QtQuick的Image类还有cache功能，我们这个例子没有那么复杂，能根据source的改变同步改变纹理内容就可以了。

由于Qt的SceneGraph的灵活性，我们需要使用许多类一起写作才能按照我们的要求创建几何体。首先我们要创建QSGGeometry::Attribute的一个数组，以便告诉SceneGraph和OpenGL，我们需要什么样的顶点缓存。注意这个数组绝对不能在栈上，因为要在updatePaintNode()函数以外用到这个数组，因此我标记为static类型。随后，我们需要QSGGeometry::AttributeSet结构的对象标识我们所需要的属性。

在设置上、下、左、右、前、后面的顶点位置坐标以及纹理坐标之后，需要对坐标进行一次适配。因为QQuickItem毕竟是一个矩形的控件，所以指定了width和height之后，boundingRect()就有相应的值了。这里需要说明的是z值。如上面几篇文章所述，要将坐标值限制在x∈[0，Screen.width]，y∈[0，Screen.height]，z∈[0，stackLayer]（其中stackLayer为Item堆叠的层数），这样才能在视口中看见。最后是Qt的source2Path()方法，用来处理路径的。

main.cpp文件也比较简单，这里我只介绍主要部分：

……
int main( int argc, char** argv )
{
	QApplication app( argc, argv );
	……
	// 注册一些类
    qmlRegisterType<TexturedCube>( "TexturedCube", 1, 0, "Cube" );
	
	QQmlApplicationEngine engine;
	engine.load( QUrl( "qrc:///qml/main.qml" ) );
	……
	return app.exec( );
}
