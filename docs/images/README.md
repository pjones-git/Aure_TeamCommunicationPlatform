# Project Images

This directory contains visual assets for the MatterMost Azure deployment project.

## Files

- **original-architecture-diagram.pdf** - Original project architecture diagram from the deployment documentation
- **architecture-diagram.png** - PNG export of the architecture diagram for README display (to be added)

## Creating architecture-diagram.png

The main README references `architecture-diagram.png`. You can create this by:

### Option 1: Export from draw.io
1. Open the PDF in draw.io (diagrams.net)
2. Export as PNG
3. Save as `architecture-diagram.png` in this directory

### Option 2: Use GitHub Mermaid Rendering
The `docs/ARCHITECTURE.md` file contains Mermaid diagrams that GitHub will automatically render. You can screenshot these rendered diagrams.

### Option 3: Convert PDF to PNG
```bash
# Using ImageMagick (if installed)
convert -density 300 original-architecture-diagram.pdf -quality 90 architecture-diagram.png

# Using macOS Preview
# Open PDF in Preview → File → Export → Format: PNG
```

## Mermaid Diagrams

The project includes Mermaid diagrams in `docs/ARCHITECTURE.md` that render automatically on GitHub:
- Network topology diagram
- Deployment sequence diagram
- Security model flowchart

These provide interactive, version-controlled architecture documentation.

## Screenshots

Consider adding the following screenshots for a complete portfolio:

1. **mattermost-interface.png** - Screenshot of the working MatterMost interface
2. **azure-resource-group.png** - Azure Portal showing all deployed resources
3. **network-topology.png** - Azure Portal network topology view
4. **nsg-rules.png** - Network Security Group rules configuration

## Notes for Recruiters

The architecture diagram demonstrates:
- Multi-tier application design
- Network security implementation
- Cloud infrastructure planning
- Professional documentation capabilities
