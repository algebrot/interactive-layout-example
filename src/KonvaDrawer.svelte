
<script>
import Konva from 'konva';
import { onMount } from 'svelte';

onMount(() => {
    var width = window.innerWidth;
    var height = window.innerHeight;
    var GUIDELINE_OFFSET = 5;
    // first we need to create a stage
    var stage = new Konva.Stage({
        container: 'konva-stage',   // id of container <div>
        width: width,
        height: height,        
    });

    // then create layer
    var layer = new Konva.Layer();
    var tooltipLayer = new Konva.Layer();

    var tooltip = new Konva.Label({
        opacity: 0.75,
        visible: false,
        listening: false
      });

    tooltip.add(
        new Konva.Tag({
          fill: 'black',
          pointerDirection: 'down',
          pointerWidth: 10,
          pointerHeight: 10,
          lineJoin: 'round',
          shadowColor: 'black',
          shadowBlur: 10,
          shadowOffset: 10,
          shadowOpacity: 0.2
        })
      );

    tooltip.add(
      new Konva.Text({
        text: '',
        fontFamily: 'Calibri',
        fontSize: 18,
        padding: 5,
        fill: 'white'
      })
    );
    
    tooltipLayer.add(tooltip);



    // first generate random rectangles
    for (var i = 0; i < 5; i++) {
      layer.add(
        new Konva.Rect({
          x: Math.random() * stage.width(),
          y: Math.random() * stage.height(),
          width: 50 + Math.random() * 50,
          height: 50 + Math.random() * 50,
          fill: Konva.Util.getRandomColor(),
          rotation: Math.random() * 360,
          draggable: true,
          name: 'object'
        })
      );
    }

    // were can we snap our objects?
    function getLineGuideStops(skipShape) {
        // we can snap to stage borders and the center of the stage
        var vertical = [0, stage.width() / 2, stage.width()];
        var horizontal = [0, stage.height() / 2, stage.height()];

        // and we snap over edges and center of each object on the canvas
        stage.find('.object').forEach(guideItem => {
          if (guideItem === skipShape) {
            return;
          }
          var box = guideItem.getClientRect();
          // and we can snap to all edges of shapes
          vertical.push([box.x, box.x + box.width, box.x + box.width / 2]);
          horizontal.push([box.y, box.y + box.height, box.y + box.height / 2]);
        });
        return {
          vertical: vertical.flat(),
          horizontal: horizontal.flat()
        };
      }

      // what points of the object will trigger to snapping?
      // it can be just center of the object
      // but we will enable all edges and center
    function getObjectSnappingEdges(node) {
        var box = node.getClientRect();
        return {
          vertical: [
            {
              guide: Math.round(box.x),
              offset: Math.round(node.x() - box.x),
              snap: 'start'
            },
            {
              guide: Math.round(box.x + box.width / 2),
              offset: Math.round(node.x() - box.x - box.width / 2),
              snap: 'center'
            },
            {
              guide: Math.round(box.x + box.width),
              offset: Math.round(node.x() - box.x - box.width),
              snap: 'end'
            }
          ],
          horizontal: [
            {
              guide: Math.round(box.y),
              offset: Math.round(node.y() - box.y),
              snap: 'start'
            },
            {
              guide: Math.round(box.y + box.height / 2),
              offset: Math.round(node.y() - box.y - box.height / 2),
              snap: 'center'
            },
            {
              guide: Math.round(box.y + box.height),
              offset: Math.round(node.y() - box.y - box.height),
              snap: 'end'
            }
          ]
        };
      }

      // find all snapping possibilities
    function getGuides(lineGuideStops, itemBounds) {
        var resultV = [];
        var resultH = [];

        lineGuideStops.vertical.forEach(lineGuide => {
          itemBounds.vertical.forEach(itemBound => {
            var diff = Math.abs(lineGuide - itemBound.guide);
            // if the distance between guild line and object snap point is close we can consider this for snapping
            if (diff < GUIDELINE_OFFSET) {
              resultV.push({
                lineGuide: lineGuide,
                diff: diff,
                snap: itemBound.snap,
                offset: itemBound.offset
              });
            }
          });
        });

        lineGuideStops.horizontal.forEach(lineGuide => {
          itemBounds.horizontal.forEach(itemBound => {
            var diff = Math.abs(lineGuide - itemBound.guide);
            if (diff < GUIDELINE_OFFSET) {
              resultH.push({
                lineGuide: lineGuide,
                diff: diff,
                snap: itemBound.snap,
                offset: itemBound.offset
              });
            }
          });
        });

      var guides = [];

        // find closest snap
      var minV = resultV.sort((a, b) => a.diff - b.diff)[0];
      var minH = resultH.sort((a, b) => a.diff - b.diff)[0];
      if (minV) {
          guides.push({
            lineGuide: minV.lineGuide,
            offset: minV.offset,
            orientation: 'V',
            snap: minV.snap
          });
        }
      if (minH) {
          guides.push({
            lineGuide: minH.lineGuide,
            offset: minH.offset,
            orientation: 'H',
            snap: minH.snap
          });
        }
      return guides;
    }

    function drawGuides(guides) {
        guides.forEach(lg => {
          if (lg.orientation === 'H') {
            var line = new Konva.Line({
              points: [-6000, lg.lineGuide, 6000, lg.lineGuide],
              stroke: 'rgb(0, 161, 255)',
              strokeWidth: 1,
              name: 'guid-line',
              dash: [4, 6]
            });
            layer.add(line);
            layer.batchDraw();
          } else if (lg.orientation === 'V') {
            var line = new Konva.Line({
              points: [lg.lineGuide, -6000, lg.lineGuide, 6000],
              stroke: 'rgb(0, 161, 255)',
              strokeWidth: 1,
              name: 'guid-line',
              dash: [4, 6]
            });
            layer.add(line);
            layer.batchDraw();
          }
        });
      }

    layer.on('dragmove', function(e) {
        // clear all previous lines on the screen
        layer.find('.guid-line').destroy();

        // find possible snapping lines
        var lineGuideStops = getLineGuideStops(e.target);
        // find snapping points of current object
        var itemBounds = getObjectSnappingEdges(e.target);

        // now find where can we snap current object
        var guides = getGuides(lineGuideStops, itemBounds);

        // do nothing of no snapping
        if (!guides.length) {
          return;
        }

    drawGuides(guides);

    // now force object position
    guides.forEach(lg => {
          switch (lg.snap) {
            case 'start': {
              switch (lg.orientation) {
                case 'V': {
                  e.target.x(lg.lineGuide + lg.offset);
                  break;
                }
                case 'H': {
                  e.target.y(lg.lineGuide + lg.offset);
                  break;
                }
              }
              break;
            }
            case 'center': {
              switch (lg.orientation) {
                case 'V': {
                  e.target.x(lg.lineGuide + lg.offset);
                  break;
                }
                case 'H': {
                  e.target.y(lg.lineGuide + lg.offset);
                  break;
                }
              }
              break;
            }
            case 'end': {
              switch (lg.orientation) {
                case 'V': {
                  e.target.x(lg.lineGuide + lg.offset);
                  break;
                }
                case 'H': {
                  e.target.y(lg.lineGuide + lg.offset);
                  break;
                }
              }
              break;
            }
          }
        });
      });

    layer.on('dragend', function(e) {
      // clear all previous lines on the screen
      layer.find('.guid-line').destroy();
      layer.batchDraw();
    });

    layer.draw();

    stage.on('click tap', function(e) {
      // if click on empty area - remove all transformers
      if (e.target === stage) {
        stage.find('Transformer').destroy();
        layer.draw();
        return;
      }
      // do nothing if clicked NOT on our rectangles
      if (!e.target.hasName('object')) {
        return;
      }
      // remove old transformers
      // TODO: we can skip it if current rect is already selected
      stage.find('Transformer').destroy();

      // create new transformer
      var tr = new Konva.Transformer();
      layer.add(tr);
      tr.attachTo(e.target);
      layer.draw();

      var mousePos = e.getStage().getPointerPosition();
      tooltip.position({
          x: mousePos.x,
          y: mousePos.y - 5
      });
      tooltip
          .getText()
          .text('object: ' + e.id() + ', color: ' + e.fill());
        tooltip.show();
        tooltipLayer.batchDraw();
      }
    
    stage.on('mouseout', function(evt) {
      tooltip.hide();
      tooltipLayer.draw();
    });

    // add the layer to the stage
    stage.add(tooltipLayer);
    stage.add(layer);


    // draw the image
    layer.draw();
});

</script>

<style>
  
</style>

<div>
    <div id="konva-stage"></div>
</div>
