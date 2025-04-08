--[[
    Delta GUI Maker for Mobile - Complete Version
    By MarKs
    
    A mobile-friendly GUI creator for Roblox with drag and drop functionality,
    script editing capabilities, template system, and code export.
    For use with Delta executor.
]]

-- Services
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")

-- Variables
local Player = Players.LocalPlayer
local Mouse = Player:GetMouse()
local Camera = workspace.CurrentCamera

-- Utility Functions
local function CreateTween(instance, properties, duration, easingStyle, easingDirection, repeatCount, reverses, delayTime)
    local tInfo = TweenInfo.new(
        duration or 0.5, 
        easingStyle or Enum.EasingStyle.Quad, 
        easingDirection or Enum.EasingDirection.Out, 
        repeatCount or 0, 
        reverses or false, 
        delayTime or 0
    )
    local tween = TweenService:Create(instance, tInfo, properties)
    return tween
end

local function RoundCorners(instance, radius)
    local uiCorner = Instance.new("UICorner")
    uiCorner.CornerRadius = UDim.new(0, radius or 8)
    uiCorner.Parent = instance
    return uiCorner
end

local function ApplyShadow(instance, shadowSize, transparency)
    local uiStroke = Instance.new("UIStroke")
    uiStroke.Thickness = shadowSize or 1
    uiStroke.Color = Color3.fromRGB(0, 0, 0)
    uiStroke.Transparency = transparency or 0.5
    uiStroke.Parent = instance
    return uiStroke
end

local function DeepCopy(original)
    local copy
    if type(original) == "table" then
        copy = {}
        for key, value in pairs(original) do
            copy[key] = DeepCopy(value)
        end
    else
        copy = original
    end
    return copy
end

local function GenerateID()
    return HttpService:GenerateGUID(false):gsub("-", ""):sub(1, 10)
end

-- Delta GUI Maker Class
local DeltaGUIMaker = {}
DeltaGUIMaker.__index = DeltaGUIMaker

-- Create new instance of GUI Maker
function DeltaGUIMaker.new()
    local self = setmetatable({}, DeltaGUIMaker)
    
    -- State variables
    self.Elements = {}
    self.SelectedElement = nil
    self.DraggingElement = nil
    self.DraggingType = nil  -- "move", "resize", "new"
    self.ResizeHandle = nil
    self.ResizeStartSize = nil
    self.ResizeStartPos = nil
    self.CopiedElement = nil
    self.UndoStack = {}
    self.RedoStack = {}
    self.SavedGUIs = {}
    self.Templates = {}
    self.ShowGrid = true
    self.GridSize = 10
    self.SnapToGrid = true
    self.CurrentScript = ""
    self.EditingScript = false
    self.CurrentColor = Color3.fromRGB(100, 100, 100) -- Default gray
    self.IsOpen = false
    self.ActiveDropdown = nil
    
    -- Create main UI
    self:CreateMainUI()
    
    return self
end
-- Create the main UI for the GUI maker
function DeltaGUIMaker:CreateMainUI()
    -- Main container
    self.MainUI = Instance.new("ScreenGui")
    self.MainUI.Name = "DeltaGUIMaker"
    self.MainUI.ResetOnSpawn = false
    self.MainUI.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    
    -- Title bar
    self.TitleBar = Instance.new("Frame")
    self.TitleBar.Name = "TitleBar"
    self.TitleBar.Size = UDim2.new(1, 0, 0.05, 0)
    self.TitleBar.Position = UDim2.new(0, 0, 0, 0)
    self.TitleBar.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    self.TitleBar.BorderSizePixel = 0
    self.TitleBar.Parent = self.MainUI
    RoundCorners(self.TitleBar, 0)
    
    -- Title text
    self.TitleText = Instance.new("TextLabel")
    self.TitleText.Name = "TitleText"
    self.TitleText.Size = UDim2.new(0.5, 0, 1, 0)
    self.TitleText.Position = UDim2.new(0, 10, 0, 0)
    self.TitleText.BackgroundTransparency = 1
    self.TitleText.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.TitleText.TextSize = 18  -- Larger for mobile
    self.TitleText.Font = Enum.Font.GothamSemibold
    self.TitleText.Text = "Delta GUI Maker - By MarKs"
    self.TitleText.TextXAlignment = Enum.TextXAlignment.Left
    self.TitleText.Parent = self.TitleBar
    
    -- Close button
    self.CloseButton = Instance.new("TextButton")
    self.CloseButton.Name = "CloseButton"
    self.CloseButton.Size = UDim2.new(0, 50, 0, 50)  -- Larger for mobile
    self.CloseButton.Position = UDim2.new(1, -50, 0, 0)
    self.CloseButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
    self.CloseButton.BorderSizePixel = 0
    self.CloseButton.Text = "X"
    self.CloseButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.CloseButton.TextSize = 22  -- Larger for mobile
    self.CloseButton.Font = Enum.Font.GothamBold
    self.CloseButton.Parent = self.TitleBar
    RoundCorners(self.CloseButton, 0)
    
    -- Work area (canvas)
    self.WorkArea = Instance.new("Frame")
    self.WorkArea.Name = "WorkArea"
    self.WorkArea.Size = UDim2.new(0.7, 0, 0.7, 0)
    self.WorkArea.Position = UDim2.new(0.15, 0, 0.2, 0)
    self.WorkArea.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    self.WorkArea.BorderSizePixel = 0
    self.WorkArea.Parent = self.MainUI
    RoundCorners(self.WorkArea, 10)  -- More rounded for mobile
    
    -- Grid container (inside work area)
    self.GridContainer = Instance.new("Frame")
    self.GridContainer.Name = "GridContainer"
    self.GridContainer.Size = UDim2.new(1, -20, 1, -20)
    self.GridContainer.Position = UDim2.new(0, 10, 0, 10)
    self.GridContainer.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    self.GridContainer.BorderSizePixel = 0
    self.GridContainer.Parent = self.WorkArea
    RoundCorners(self.GridContainer, 8)  -- More rounded for mobile
    
    -- Toolbar
    self.Toolbar = Instance.new("Frame")
    self.Toolbar.Name = "Toolbar"
    self.Toolbar.Size = UDim2.new(1, 0, 0.06, 0)  -- Taller for mobile
    self.Toolbar.Position = UDim2.new(0, 0, 0.05, 0)
    self.Toolbar.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    self.Toolbar.BorderSizePixel = 0
    self.Toolbar.Parent = self.MainUI
    
    -- Create toolbar buttons
    self:CreateToolbarButtons()
    
    -- Properties panel
    self.PropertiesPanel = Instance.new("Frame")
    self.PropertiesPanel.Name = "PropertiesPanel"
    self.PropertiesPanel.Size = UDim2.new(0.25, 0, 0.9, 0)
    self.PropertiesPanel.Position = UDim2.new(0.75, 0, 0.1, 0)
    self.PropertiesPanel.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    self.PropertiesPanel.BorderSizePixel = 0
    self.PropertiesPanel.Visible = true
    self.PropertiesPanel.Parent = self.MainUI
    RoundCorners(self.PropertiesPanel, 10)  -- More rounded for mobile
    
    -- Properties panel header
    self.PropertiesPanelHeader = Instance.new("TextLabel")
    self.PropertiesPanelHeader.Name = "Header"
    self.PropertiesPanelHeader.Size = UDim2.new(1, 0, 0.05, 0)
    self.PropertiesPanelHeader.Position = UDim2.new(0, 0, 0, 0)
    self.PropertiesPanelHeader.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    self.PropertiesPanelHeader.BorderSizePixel = 0
    self.PropertiesPanelHeader.Text = "Properties"
    self.PropertiesPanelHeader.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.PropertiesPanelHeader.TextSize = 18  -- Larger for mobile
    self.PropertiesPanelHeader.Font = Enum.Font.GothamSemibold
    self.PropertiesPanelHeader.Parent = self.PropertiesPanel
    RoundCorners(self.PropertiesPanelHeader, 10)  -- More rounded for mobile
    
    -- Properties scroll frame
    self.PropertiesScroll = Instance.new("ScrollingFrame")
    self.PropertiesScroll.Name = "PropertiesScroll"
    self.PropertiesScroll.Size = UDim2.new(1, -10, 0.95, -10)
    self.PropertiesScroll.Position = UDim2.new(0, 5, 0.05, 5)
    self.PropertiesScroll.BackgroundTransparency = 1
    self.PropertiesScroll.BorderSizePixel = 0
    self.PropertiesScroll.ScrollBarThickness = 6  -- Thicker for mobile
    self.PropertiesScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
    self.PropertiesScroll.ScrollingDirection = Enum.ScrollingDirection.Y
    self.PropertiesScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
    self.PropertiesScroll.Parent = self.PropertiesPanel
    
    -- Properties layout
    self.PropertiesLayout = Instance.new("UIListLayout")
    self.PropertiesLayout.SortOrder = Enum.SortOrder.LayoutOrder
    self.PropertiesLayout.Padding = UDim.new(0, 8)  -- More padding for mobile
    self.PropertiesLayout.Parent = self.PropertiesScroll
    
    -- Element palette
    self.ElementPalette = Instance.new("Frame")
    self.ElementPalette.Name = "ElementPalette"
    self.ElementPalette.Size = UDim2.new(0.25, 0, 0.9, 0)
    self.ElementPalette.Position = UDim2.new(0, 0, 0.1, 0)
    self.ElementPalette.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    self.ElementPalette.BorderSizePixel = 0
    self.ElementPalette.Visible = true
    self.ElementPalette.Parent = self.MainUI
    RoundCorners(self.ElementPalette, 10)  -- More rounded for mobile
    
    -- Element palette header
    self.ElementPaletteHeader = Instance.new("TextLabel")
    self.ElementPaletteHeader.Name = "Header"
    self.ElementPaletteHeader.Size = UDim2.new(1, 0, 0.05, 0)
    self.ElementPaletteHeader.Position = UDim2.new(0, 0, 0, 0)
    self.ElementPaletteHeader.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    self.ElementPaletteHeader.BorderSizePixel = 0
    self.ElementPaletteHeader.Text = "Elements"
    self.ElementPaletteHeader.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.ElementPaletteHeader.TextSize = 18  -- Larger for mobile
    self.ElementPaletteHeader.Font = Enum.Font.GothamSemibold
    self.ElementPaletteHeader.Parent = self.ElementPalette
    RoundCorners(self.ElementPaletteHeader, 10)  -- More rounded for mobile
    
    -- Element palette scroll
    self.ElementPaletteScroll = Instance.new("ScrollingFrame")
    self.ElementPaletteScroll.Name = "ElementScroll"
    self.ElementPaletteScroll.Size = UDim2.new(1, -10, 0.95, -10)
    self.ElementPaletteScroll.Position = UDim2.new(0, 5, 0.05, 5)
    self.ElementPaletteScroll.BackgroundTransparency = 1
    self.ElementPaletteScroll.BorderSizePixel = 0
    self.ElementPaletteScroll.ScrollBarThickness = 6  -- Thicker for mobile
    self.ElementPaletteScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
    self.ElementPaletteScroll.ScrollingDirection = Enum.ScrollingDirection.Y
    self.ElementPaletteScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
    self.ElementPaletteScroll.Parent = self.ElementPalette
    
    -- Element palette layout
    self.ElementPaletteLayout = Instance.new("UIListLayout")
    self.ElementPaletteLayout.SortOrder = Enum.SortOrder.LayoutOrder
    self.ElementPaletteLayout.Padding = UDim.new(0, 8)  -- More padding for mobile
    self.ElementPaletteLayout.Parent = self.ElementPaletteScroll
    
    -- Create element buttons
    self:CreateElementButtons()
    
    -- Template system panel
    self.TemplatePanel = Instance.new("Frame")
    self.TemplatePanel.Name = "TemplatePanel"
    self.TemplatePanel.Size = UDim2.new(0.8, 0, 0.8, 0)
    self.TemplatePanel.Position = UDim2.new(0.1, 0, 0.1, 0)
    self.TemplatePanel.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    self.TemplatePanel.BorderSizePixel = 0
    self.TemplatePanel.Visible = false
    self.TemplatePanel.Parent = self.MainUI
    RoundCorners(self.TemplatePanel, 10)  -- More rounded for mobile
    
    -- Template panel header
    self.TemplatePanelHeader = Instance.new("TextLabel")
    self.TemplatePanelHeader.Name = "Header"
    self.TemplatePanelHeader.Size = UDim2.new(1, 0, 0.05, 0)
    self.TemplatePanelHeader.Position = UDim2.new(0, 0, 0, 0)
    self.TemplatePanelHeader.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    self.TemplatePanelHeader.BorderSizePixel = 0
    self.TemplatePanelHeader.Text = "Templates"
    self.TemplatePanelHeader.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.TemplatePanelHeader.TextSize = 18  -- Larger for mobile
    self.TemplatePanelHeader.Font = Enum.Font.GothamSemibold
    self.TemplatePanelHeader.Parent = self.TemplatePanel
    RoundCorners(self.TemplatePanelHeader, 10)  -- More rounded for mobile
    
    -- Template search bar
    self.TemplateSearchBar = Instance.new("TextBox")
    self.TemplateSearchBar.Name = "SearchBar"
    self.TemplateSearchBar.Size = UDim2.new(0.7, 0, 0.05, 0)
    self.TemplateSearchBar.Position = UDim2.new(0.05, 0, 0.06, 0)
    self.TemplateSearchBar.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    self.TemplateSearchBar.BorderSizePixel = 0
    self.TemplateSearchBar.PlaceholderText = "Search templates..."
    self.TemplateSearchBar.Text = ""
    self.TemplateSearchBar.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.TemplateSearchBar.PlaceholderColor3 = Color3.fromRGB(180, 180, 180)
    self.TemplateSearchBar.TextSize = 16  -- Larger for mobile
    self.TemplateSearchBar.Font = Enum.Font.Gotham
    self.TemplateSearchBar.Parent = self.TemplatePanel
    RoundCorners(self.TemplateSearchBar, 8)

    -- Template search button
    self.TemplateSearchButton = Instance.new("TextButton")
    self.TemplateSearchButton.Name = "SearchButton"
    self.TemplateSearchButton.Size = UDim2.new(0.15, 0, 0.05, 0)
    self.TemplateSearchButton.Position = UDim2.new(0.8, 0, 0.06, 0)
    self.TemplateSearchButton.BackgroundColor3 = Color3.fromRGB(60, 120, 200)
    self.TemplateSearchButton.BorderSizePixel = 0
    self.TemplateSearchButton.Text = "Search"
    self.TemplateSearchButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.TemplateSearchButton.TextSize = 16  -- Larger for mobile
    self.TemplateSearchButton.Font = Enum.Font.GothamSemibold
    self.TemplateSearchButton.Parent = self.TemplatePanel
    RoundCorners(self.TemplateSearchButton, 8)
    
    -- Template list container
    self.TemplateListContainer = Instance.new("ScrollingFrame")
    self.TemplateListContainer.Name = "TemplateList"
    self.TemplateListContainer.Size = UDim2.new(0.9, 0, 0.75, 0)
    self.TemplateListContainer.Position = UDim2.new(0.05, 0, 0.12, 0)
    self.TemplateListContainer.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    self.TemplateListContainer.BorderSizePixel = 0
    self.TemplateListContainer.ScrollBarThickness = 6  -- Thicker for mobile
    self.TemplateListContainer.CanvasSize = UDim2.new(0, 0, 0, 0)
    self.TemplateListContainer.ScrollingDirection = Enum.ScrollingDirection.Y
    self.TemplateListContainer.AutomaticCanvasSize = Enum.AutomaticSize.Y
    self.TemplateListContainer.Parent = self.TemplatePanel
    RoundCorners(self.TemplateListContainer, 8)
    
    -- Template list layout
    self.TemplateListLayout = Instance.new("UIListLayout")
    self.TemplateListLayout.SortOrder = Enum.SortOrder.Name
    self.TemplateListLayout.Padding = UDim.new(0, 8)  -- More padding for mobile
    self.TemplateListLayout.Parent = self.TemplateListContainer
    
    -- Template padding
    local templatePadding = Instance.new("UIPadding")
    templatePadding.PaddingTop = UDim.new(0, 5)
    templatePadding.PaddingBottom = UDim.new(0, 5)
    templatePadding.PaddingLeft = UDim.new(0, 5)
    templatePadding.PaddingRight = UDim.new(0, 5)
    templatePadding.Parent = self.TemplateListContainer
    
    -- Create template button
    self.CreateTemplateButton = Instance.new("TextButton")
    self.CreateTemplateButton.Name = "CreateTemplateButton"
    self.CreateTemplateButton.Size = UDim2.new(0.25, 0, 0.06, 0)
    self.CreateTemplateButton.Position = UDim2.new(0.125, 0, 0.89, 0)
    self.CreateTemplateButton.BackgroundColor3 = Color3.fromRGB(60, 150, 80)
    self.CreateTemplateButton.BorderSizePixel = 0
    self.CreateTemplateButton.Text = "Create Template"
    self.CreateTemplateButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.CreateTemplateButton.TextSize = 16  -- Larger for mobile
    self.CreateTemplateButton.Font = Enum.Font.GothamSemibold
    self.CreateTemplateButton.Parent = self.TemplatePanel
    RoundCorners(self.CreateTemplateButton, 8)
    
    -- Close template panel button
    self.CloseTemplateButton = Instance.new("TextButton")
    self.CloseTemplateButton.Name = "CloseTemplateButton"
    self.CloseTemplateButton.Size = UDim2.new(0.15, 0, 0.06, 0)
    self.CloseTemplateButton.Position = UDim2.new(0.725, 0, 0.89, 0)
    self.CloseTemplateButton.BackgroundColor3 = Color3.fromRGB(200, 60, 60)
    self.CloseTemplateButton.BorderSizePixel = 0
    self.CloseTemplateButton.Text = "Close"
    self.CloseTemplateButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.CloseTemplateButton.TextSize = 16  -- Larger for mobile
    self.CloseTemplateButton.Font = Enum.Font.GothamSemibold
    self.CloseTemplateButton.Parent = self.TemplatePanel
    RoundCorners(self.CloseTemplateButton, 8)
    
    -- Template creation panel
    self.TemplateCreationPanel = Instance.new("Frame")
    self.TemplateCreationPanel.Name = "TemplateCreationPanel"
    self.TemplateCreationPanel.Size = UDim2.new(0.6, 0, 0.5, 0)
    self.TemplateCreationPanel.Position = UDim2.new(0.2, 0, 0.25, 0)
    self.TemplateCreationPanel.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    self.TemplateCreationPanel.BorderSizePixel = 0
    self.TemplateCreationPanel.Visible = false
    self.TemplateCreationPanel.Parent = self.MainUI
    self.TemplateCreationPanel.ZIndex = 100  -- Above everything else
    RoundCorners(self.TemplateCreationPanel, 10)  -- More rounded for mobile
    
    -- Template creation header
    self.TemplateCreationHeader = Instance.new("TextLabel")
    self.TemplateCreationHeader.Name = "Header"
    self.TemplateCreationHeader.Size = UDim2.new(1, 0, 0.1, 0)
    self.TemplateCreationHeader.Position = UDim2.new(0, 0, 0, 0)
    self.TemplateCreationHeader.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    self.TemplateCreationHeader.BorderSizePixel = 0
    self.TemplateCreationHeader.Text = "Create New Template"
    self.TemplateCreationHeader.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.TemplateCreationHeader.TextSize = 18  -- Larger for mobile
    self.TemplateCreationHeader.Font = Enum.Font.GothamSemibold
    self.TemplateCreationHeader.ZIndex = 100
    self.TemplateCreationHeader.Parent = self.TemplateCreationPanel
    RoundCorners(self.TemplateCreationHeader, 10)  -- More rounded for mobile
    
    -- Template name label
    self.TemplateNameLabel = Instance.new("TextLabel")
    self.TemplateNameLabel.Name = "NameLabel"
    self.TemplateNameLabel.Size = UDim2.new(0.9, 0, 0.08, 0)
    self.TemplateNameLabel.Position = UDim2.new(0.05, 0, 0.15, 0)
    self.TemplateNameLabel.BackgroundTransparency = 1
    self.TemplateNameLabel.Text = "Template Name:"
    self.TemplateNameLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.TemplateNameLabel.TextSize = 16  -- Larger for mobile
    self.TemplateNameLabel.Font = Enum.Font.Gotham
    self.TemplateNameLabel.TextXAlignment = Enum.TextXAlignment.Left
    self.TemplateNameLabel.ZIndex = 100
    self.TemplateNameLabel.Parent = self.TemplateCreationPanel
    
    -- Template name input
    self.TemplateNameInput = Instance.new("TextBox")
    self.TemplateNameInput.Name = "NameInput"
    self.TemplateNameInput.Size = UDim2.new(0.9, 0, 0.08, 0)
    self.TemplateNameInput.Position = UDim2.new(0.05, 0, 0.25, 0)
    self.TemplateNameInput.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    self.TemplateNameInput.BorderSizePixel = 0
    self.TemplateNameInput.Text = ""
    self.TemplateNameInput.PlaceholderText = "Enter template name..."
    self.TemplateNameInput.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.TemplateNameInput.PlaceholderColor3 = Color3.fromRGB(180, 180, 180)
    self.TemplateNameInput.TextSize = 16  -- Larger for mobile
    self.TemplateNameInput.Font = Enum.Font.Gotham
    self.TemplateNameInput.ZIndex = 100
    self.TemplateNameInput.Parent = self.TemplateCreationPanel
    RoundCorners(self.TemplateNameInput, 8)
    
    -- Template description label
    self.TemplateDescLabel = Instance.new("TextLabel")
    self.TemplateDescLabel.Name = "DescLabel"
    self.TemplateDescLabel.Size = UDim2.new(0.9, 0, 0.08, 0)
    self.TemplateDescLabel.Position = UDim2.new(0.05, 0, 0.35, 0)
    self.TemplateDescLabel.BackgroundTransparency = 1
    self.TemplateDescLabel.Text = "Description:"
    self.TemplateDescLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.TemplateDescLabel.TextSize = 16  -- Larger for mobile
    self.TemplateDescLabel.Font = Enum.Font.Gotham
    self.TemplateDescLabel.TextXAlignment = Enum.TextXAlignment.Left
    self.TemplateDescLabel.ZIndex = 100
    self.TemplateDescLabel.Parent = self.TemplateCreationPanel
    
    -- Template description input
    self.TemplateDescInput = Instance.new("TextBox")
    self.TemplateDescInput.Name = "DescInput"
    self.TemplateDescInput.Size = UDim2.new(0.9, 0, 0.2, 0)
    self.TemplateDescInput.Position = UDim2.new(0.05, 0, 0.45, 0)
    self.TemplateDescInput.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    self.TemplateDescInput.BorderSizePixel = 0
    self.TemplateDescInput.Text = ""
    self.TemplateDescInput.PlaceholderText = "Enter description..."
    self.TemplateDescInput.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.TemplateDescInput.PlaceholderColor3 = Color3.fromRGB(180, 180, 180)
    self.TemplateDescInput.TextSize = 16  -- Larger for mobile
    self.TemplateDescInput.Font = Enum.Font.Gotham
    self.TemplateDescInput.TextWrapped = true
    self.TemplateDescInput.MultiLine = true
    self.TemplateDescInput.ZIndex = 100
    self.TemplateDescInput.Parent = self.TemplateCreationPanel
    RoundCorners(self.TemplateDescInput, 8)
    
    -- Save template button
    self.SaveTemplateButton = Instance.new("TextButton")
    self.SaveTemplateButton.Name = "SaveButton"
    self.SaveTemplateButton.Size = UDim2.new(0.3, 0, 0.1, 0)
    self.SaveTemplateButton.Position = UDim2.new(0.2, 0, 0.8, 0)
    self.SaveTemplateButton.BackgroundColor3 = Color3.fromRGB(60, 150, 80)
    self.SaveTemplateButton.BorderSizePixel = 0
    self.SaveTemplateButton.Text = "Save"
    self.SaveTemplateButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.SaveTemplateButton.TextSize = 18  -- Larger for mobile
    self.SaveTemplateButton.Font = Enum.Font.GothamSemibold
    self.SaveTemplateButton.ZIndex = 100
    self.SaveTemplateButton.Parent = self.TemplateCreationPanel
    RoundCorners(self.SaveTemplateButton, 8)
    
    -- Cancel template button
    self.CancelTemplateButton = Instance.new("TextButton")
    self.CancelTemplateButton.Name = "CancelButton"
    self.CancelTemplateButton.Size = UDim2.new(0.3, 0, 0.1, 0)
    self.CancelTemplateButton.Position = UDim2.new(0.55, 0, 0.8, 0)
    self.CancelTemplateButton.BackgroundColor3 = Color3.fromRGB(200, 60, 60)
    self.CancelTemplateButton.BorderSizePixel = 0
    self.CancelTemplateButton.Text = "Cancel"
    self.CancelTemplateButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.CancelTemplateButton.TextSize = 18  -- Larger for mobile
    self.CancelTemplateButton.Font = Enum.Font.GothamSemibold
    self.CancelTemplateButton.ZIndex = 100
    self.CancelTemplateButton.Parent = self.TemplateCreationPanel
    RoundCorners(self.CancelTemplateButton, 8)
    
    -- Code export window
    self.CodeExportWindow = Instance.new("Frame")
    self.CodeExportWindow.Name = "CodeExportWindow"
    self.CodeExportWindow.Size = UDim2.new(0.8, 0, 0.8, 0)
    self.CodeExportWindow.Position = UDim2.new(0.1, 0, 0.1, 0)
    self.CodeExportWindow.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    self.CodeExportWindow.BorderSizePixel = 0
    self.CodeExportWindow.Visible = false
    self.CodeExportWindow.Parent = self.MainUI
    RoundCorners(self.CodeExportWindow, 10)  -- More rounded for mobile
    
    -- Code export header
    self.CodeExportHeader = Instance.new("TextLabel")
    self.CodeExportHeader.Name = "Header"
    self.CodeExportHeader.Size = UDim2.new(1, 0, 0.05, 0)
    self.CodeExportHeader.Position = UDim2.new(0, 0, 0, 0)
    self.CodeExportHeader.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    self.CodeExportHeader.BorderSizePixel = 0
    self.CodeExportHeader.Text = "Generated Code"
    self.CodeExportHeader.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.CodeExportHeader.TextSize = 18  -- Larger for mobile
    self.CodeExportHeader.Font = Enum.Font.GothamSemibold
    self.CodeExportHeader.Parent = self.CodeExportWindow
    RoundCorners(self.CodeExportHeader, 10)  -- More rounded for mobile
    
    -- Code text box
    self.CodeTextBox = Instance.new("TextBox")
    self.CodeTextBox.Name = "CodeTextBox"
    self.CodeTextBox.Size = UDim2.new(1, -20, 0.85, 0)
    self.CodeTextBox.Position = UDim2.new(0, 10, 0.05, 10)
    self.CodeTextBox.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    self.CodeTextBox.BorderSizePixel = 0
    self.CodeTextBox.TextColor3 = Color3.fromRGB(220, 220, 220)
    self.CodeTextBox.TextSize = 16  -- Larger for mobile
    self.CodeTextBox.Font = Enum.Font.Code
    self.CodeTextBox.Text = "-- Code will appear here"
    self.CodeTextBox.TextXAlignment = Enum.TextXAlignment.Left
    self.CodeTextBox.TextYAlignment = Enum.TextYAlignment.Top
    self.CodeTextBox.TextWrapped = false
    self.CodeTextBox.ClearTextOnFocus = false
    self.CodeTextBox.MultiLine = true
    self.CodeTextBox.Parent = self.CodeExportWindow
    RoundCorners(self.CodeTextBox, 8)
    
    -- Copy code button
    self.CopyCodeButton = Instance.new("TextButton")
    self.CopyCodeButton.Name = "CopyButton"
    self.CopyCodeButton.Size = UDim2.new(0.25, 0, 0.08, 0)  -- Larger for mobile
    self.CopyCodeButton.Position = UDim2.new(0.375, 0, 0.91, 0)
    self.CopyCodeButton.BackgroundColor3 = Color3.fromRGB(60, 120, 200)
    self.CopyCodeButton.BorderSizePixel = 0
    self.CopyCodeButton.Text = "Copy Code"
    self.CopyCodeButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.CopyCodeButton.TextSize = 16  -- Larger for mobile
    self.CopyCodeButton.Font = Enum.Font.GothamSemibold
    self.CopyCodeButton.Parent = self.CodeExportWindow
    RoundCorners(self.CopyCodeButton, 8)
    
    -- Close export button
    self.CloseExportButton = Instance.new("TextButton")
    self.CloseExportButton.Name = "CloseButton"
    self.CloseExportButton.Size = UDim2.new(0.15, 0, 0.08, 0)  -- Larger for mobile
    self.CloseExportButton.Position = UDim2.new(0.8, 0, 0.91, 0)
    self.CloseExportButton.BackgroundColor3 = Color3.fromRGB(200, 60, 60)
    self.CloseExportButton.BorderSizePixel = 0
    self.CloseExportButton.Text = "Close"
    self.CloseExportButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.CloseExportButton.TextSize = 16  -- Larger for mobile
    self.CloseExportButton.Font = Enum.Font.GothamSemibold
    self.CloseExportButton.Parent = self.CodeExportWindow
    RoundCorners(self.CloseExportButton, 8)
    
    -- Script editor window
    self.ScriptEditorWindow = Instance.new("Frame")
    self.ScriptEditorWindow.Name = "ScriptEditorWindow"
    self.ScriptEditorWindow.Size = UDim2.new(0.8, 0, 0.8, 0)
    self.ScriptEditorWindow.Position = UDim2.new(0.1, 0, 0.1, 0)
    self.ScriptEditorWindow.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    self.ScriptEditorWindow.BorderSizePixel = 0
    self.ScriptEditorWindow.Visible = false
    self.ScriptEditorWindow.Parent = self.MainUI
    RoundCorners(self.ScriptEditorWindow, 10)  -- More rounded for mobile
    
    -- Script editor header
    self.ScriptEditorHeader = Instance.new("TextLabel")
    self.ScriptEditorHeader.Name = "Header"
    self.ScriptEditorHeader.Size = UDim2.new(1, 0, 0.05, 0)
    self.ScriptEditorHeader.Position = UDim2.new(0, 0, 0, 0)
    self.ScriptEditorHeader.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    self.ScriptEditorHeader.BorderSizePixel = 0
    self.ScriptEditorHeader.Text = "Script Editor"
    self.ScriptEditorHeader.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.ScriptEditorHeader.TextSize = 18  -- Larger for mobile
    self.ScriptEditorHeader.Font = Enum.Font.GothamSemibold
    self.ScriptEditorHeader.Parent = self.ScriptEditorWindow
    RoundCorners(self.ScriptEditorHeader, 10)  -- More rounded for mobile
    
    -- Script text box
    self.ScriptTextBox = Instance.new("TextBox")
    self.ScriptTextBox.Name = "ScriptTextBox"
    self.ScriptTextBox.Size = UDim2.new(1, -20, 0.85, 0)
    self.ScriptTextBox.Position = UDim2.new(0, 10, 0.05, 10)
    self.ScriptTextBox.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    self.ScriptTextBox.BorderSizePixel = 0
    self.ScriptTextBox.TextColor3 = Color3.fromRGB(220, 220, 220)
    self.ScriptTextBox.TextSize = 16  -- Larger for mobile
    self.ScriptTextBox.Font = Enum.Font.Code
    self.ScriptTextBox.Text = "-- Write your script here\n-- Example:\n-- local button = script.Parent\n-- button.MouseButton1Click:Connect(function()\n--     print('Button clicked!')\n-- end)"
    self.ScriptTextBox.TextXAlignment = Enum.TextXAlignment.Left
    self.ScriptTextBox.TextYAlignment = Enum.TextYAlignment.Top
    self.ScriptTextBox.TextWrapped = false
    self.ScriptTextBox.ClearTextOnFocus = false
    self.ScriptTextBox.MultiLine = true
    self.ScriptTextBox.Parent = self.ScriptEditorWindow
    RoundCorners(self.ScriptTextBox, 8)
    
    -- Save script button
    self.SaveScriptButton = Instance.new("TextButton")
    self.SaveScriptButton.Name = "SaveButton"
    self.SaveScriptButton.Size = UDim2.new(0.2, 0, 0.08, 0)  -- Larger for mobile
    self.SaveScriptButton.Position = UDim2.new(0.4, 0, 0.91, 0)
    self.SaveScriptButton.BackgroundColor3 = Color3.fromRGB(60, 150, 80)
    self.SaveScriptButton.BorderSizePixel = 0
    self.SaveScriptButton.Text = "Save Script"
    self.SaveScriptButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.SaveScriptButton.TextSize = 16  -- Larger for mobile
    self.SaveScriptButton.Font = Enum.Font.GothamSemibold
    self.SaveScriptButton.Parent = self.ScriptEditorWindow
    RoundCorners(self.SaveScriptButton, 8)
    
    -- Close script editor button
    self.CloseScriptButton = Instance.new("TextButton")
    self.CloseScriptButton.Name = "CloseButton"
    self.CloseScriptButton.Size = UDim2.new(0.15, 0, 0.08, 0)  -- Larger for mobile
    self.CloseScriptButton.Position = UDim2.new(0.8, 0, 0.91, 0)
    self.CloseScriptButton.BackgroundColor3 = Color3.fromRGB(200, 60, 60)
    self.CloseScriptButton.BorderSizePixel = 0
    self.CloseScriptButton.Text = "Close"
    self.CloseScriptButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.CloseScriptButton.TextSize = 16  -- Larger for mobile
    self.CloseScriptButton.Font = Enum.Font.GothamSemibold
    self.CloseScriptButton.Parent = self.ScriptEditorWindow
    RoundCorners(self.CloseScriptButton, 8)
    
    -- Register events
    self:RegisterEvents()
end

-- Template System Functions
function DeltaGUIMaker:ShowTemplatePanel()
    self.TemplatePanel.Visible = true
    self:RefreshTemplateList()
end

function DeltaGUIMaker:HideTemplatePanel()
    self.TemplatePanel.Visible = false
end

function DeltaGUIMaker:ShowTemplateCreationPanel()
    self.TemplateCreationPanel.Visible = true
    self.TemplateNameInput.Text = ""
    self.TemplateDescInput.Text = ""
end

function DeltaGUIMaker:HideTemplateCreationPanel()
    self.TemplateCreationPanel.Visible = false
end

function DeltaGUIMaker:SaveTemplate()
    local templateName = self.TemplateNameInput.Text
    if templateName == "" then
        self:ShowNotification("Please enter a template name.", Color3.fromRGB(255, 80, 80))
        return
    end
    
    -- Check if template with same name exists
    for _, template in pairs(self.Templates) do
        if template.Name == templateName and template.Creator ~= Player.Name then
            self:ShowNotification("A template with this name already exists.", Color3.fromRGB(255, 80, 80))
            return
        end
    end
    
    -- Create template data
    local templateData = {
        Name = templateName,
        Description = self.TemplateDescInput.Text,
        Creator = Player.Name,
        Elements = DeepCopy(self.Elements),
        CreationDate = os.date("%Y-%m-%d"),
        ID = GenerateID()
    }
    
    -- Add/update template
    local updated = false
    for i, template in pairs(self.Templates) do
        if template.Name == templateName and template.Creator == Player.Name then
            self.Templates[i] = templateData
            updated = true
            break
        end
    end
    
    if not updated then
        table.insert(self.Templates, templateData)
    end
    
    self:SaveTemplatesData()
    self:RefreshTemplateList()
    self:HideTemplateCreationPanel()
    self:ShowNotification("Template saved successfully!", Color3.fromRGB(80, 255, 120))
end

function DeltaGUIMaker:DeleteTemplate(templateID)
    for i, template in pairs(self.Templates) do
        if template.ID == templateID then
            if template.Creator == Player.Name then
                table.remove(self.Templates, i)
                self:SaveTemplatesData()
                self:RefreshTemplateList()
                self:ShowNotification("Template deleted.", Color3.fromRGB(255, 200, 80))
                return
            else
                self:ShowNotification("You can only delete your own templates.", Color3.fromRGB(255, 80, 80))
                return
            end
        end
    end
end

function DeltaGUIMaker:LoadTemplate(templateID)
    for _, template in pairs(self.Templates) do
        if template.ID == templateID then
            -- Confirm before loading
            if #self.Elements > 0 then
                -- Create confirmation dialog (simplified for mobile)
                local confirmFrame = Instance.new("Frame")
                confirmFrame.Size = UDim2.new(0.4, 0, 0.2, 0)
                confirmFrame.Position = UDim2.new(0.3, 0, 0.4, 0)
                confirmFrame.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
                confirmFrame.BorderSizePixel = 0
                confirmFrame.ZIndex = 1000
                confirmFrame.Parent = self.MainUI
                RoundCorners(confirmFrame, 10)
                
                local confirmText = Instance.new("TextLabel")
                confirmText.Size = UDim2.new(1, 0, 0.5, 0)
                confirmText.Position = UDim2.new(0, 0, 0, 0)
                confirmText.BackgroundTransparency = 1
                confirmText.Text = "Load template? Current work will be lost."
                confirmText.TextColor3 = Color3.fromRGB(255, 255, 255)
                confirmText.TextSize = 18  -- Larger for mobile
                confirmText.Font = Enum.Font.GothamSemibold
                confirmText.ZIndex = 1001
                confirmText.Parent = confirmFrame
                
                local confirmButton = Instance.new("TextButton")
                confirmButton.Size = UDim2.new(0.4, 0, 0.3, 0)
                confirmButton.Position = UDim2.new(0.1, 0, 0.6, 0)
                confirmButton.BackgroundColor3 = Color3.fromRGB(60, 150, 80)
                confirmButton.BorderSizePixel = 0
                confirmButton.Text = "Load"
                confirmButton.TextColor3 = Color3.fromRGB(255, 255, 255)
                confirmButton.TextSize = 16  -- Larger for mobile
                confirmButton.Font = Enum.Font.GothamSemibold
                confirmButton.ZIndex = 1001
                confirmButton.Parent = confirmFrame
                RoundCorners(confirmButton, 8)
                
                local cancelButton = Instance.new("TextButton")
                cancelButton.Size = UDim2.new(0.4, 0, 0.3, 0)
                cancelButton.Position = UDim2.new(0.55, 0, 0.6, 0)
                cancelButton.BackgroundColor3 = Color3.fromRGB(200, 60, 60)
                cancelButton.BorderSizePixel = 0
                cancelButton.Text = "Cancel"
                cancelButton.TextColor3 = Color3.fromRGB(255, 255, 255)
                cancelButton.TextSize = 16  -- Larger for mobile
                cancelButton.Font = Enum.Font.GothamSemibold
                cancelButton.ZIndex = 1001
                cancelButton.Parent = confirmFrame
                RoundCorners(cancelButton, 8)
                
                confirmButton.MouseButton1Click:Connect(function()
                    -- Clear current elements
                    for _, element in pairs(self.Elements) do
                        element.Instance:Destroy()
                    end
                    
                    self.Elements = {}
                    
                    -- Load template elements
                    for _, elementData in pairs(template.Elements) do
                        local newElementData = DeepCopy(elementData)
                        newElementData.Instance = self:CreateElementInstance(newElementData)
                        table.insert(self.Elements, newElementData)
                    end
                    
                    confirmFrame:Destroy()
                    self:HideTemplatePanel()
                    self:ShowNotification("Template loaded: " .. template.Name, Color3.fromRGB(80, 255, 120))
                end)
                
                cancelButton.MouseButton1Click:Connect(function()
                    confirmFrame:Destroy()
                end)
            else
                -- No existing elements, load directly
                for _, elementData in pairs(template.Elements) do
                    local newElementData = DeepCopy(elementData)
                    newElementData.Instance = self:CreateElementInstance(newElementData)
                    table.insert(self.Elements, newElementData)
                end
                
                self:HideTemplatePanel()
                self:ShowNotification("Template loaded: " .. template.Name, Color3.fromRGB(80, 255, 120))
            end
            
            return
        end
    end
    
    self:ShowNotification("Template not found.", Color3.fromRGB(255, 80, 80))
end

function DeltaGUIMaker:CreateTemplateEntry(template)
    local entry = Instance.new("Frame")
    entry.Name = template.Name
    entry.Size = UDim2.new(1, -10, 0, 120)  -- Taller for mobile
    entry.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
    entry.BorderSizePixel = 0
    entry.Parent = self.TemplateListContainer
    RoundCorners(entry, 8)
    
    -- Template title
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -20, 0, 25)  -- Taller for mobile
    title.Position = UDim2.new(0, 10, 0, 5)
    title.BackgroundTransparency = 1
    title.Text = template.Name
    title.TextColor3 = Color3.fromRGB(255, 255, 255)
    title.TextSize = 18  -- Larger for mobile
    title.Font = Enum.Font.GothamBold
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = entry
    
    -- Creator label
    local creator = Instance.new("TextLabel")
    creator.Size = UDim2.new(1, -20, 0, 20)
    creator.Position = UDim2.new(0, 10, 0, 30)
    creator.BackgroundTransparency = 1
    creator.Text = "By: " .. template.Creator
    creator.TextColor3 = Color3.fromRGB(200, 200, 200)
    creator.TextSize = 14
    creator.Font = Enum.Font.Gotham
    creator.TextXAlignment = Enum.TextXAlignment.Left
    creator.Parent = entry
    
    -- Description
    local description = Instance.new("TextLabel")
    description.Size = UDim2.new(1, -20, 0, 40)
    description.Position = UDim2.new(0, 10, 0, 50)
    description.BackgroundTransparency = 1
    description.Text = template.Description ~= "" and template.Description or "No description provided."
    description.TextColor3 = Color3.fromRGB(180, 180, 180)
    description.TextSize = 14
    description.Font = Enum.Font.Gotham
    description.TextXAlignment = Enum.TextXAlignment.Left
    description.TextYAlignment = Enum.TextYAlignment.Top
    description.TextWrapped = true
    description.Parent = entry
    
    -- Load button
    local loadButton = Instance.new("TextButton")
    loadButton.Size = UDim2.new(0.2, 0, 0, 30)  -- Taller for mobile
    loadButton.Position = UDim2.new(0.05, 0, 0, 85)
    loadButton.BackgroundColor3 = Color3.fromRGB(60, 120, 200)
    loadButton.BorderSizePixel = 0
    loadButton.Text = "Load"
    loadButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    loadButton.TextSize = 16  -- Larger for mobile
    loadButton.Font = Enum.Font.GothamSemibold
    loadButton.Parent = entry
    RoundCorners(loadButton, 8)
    
    -- Delete button (only for owner)
    if template.Creator == Player.Name then
        local deleteButton = Instance.new("TextButton")
        deleteButton.Size = UDim2.new(0.2, 0, 0, 30)  -- Taller for mobile
        deleteButton.Position = UDim2.new(0.75, 0, 0, 85)
        deleteButton.BackgroundColor3 = Color3.fromRGB(200, 60, 60)
        deleteButton.BorderSizePixel = 0
        deleteButton.Text = "Delete"
        deleteButton.TextColor3 = Color3.fromRGB(255, 255, 255)
        deleteButton.TextSize = 16  -- Larger for mobile
        deleteButton.Font = Enum.Font.GothamSemibold
        deleteButton.Parent = entry
        RoundCorners(deleteButton, 8)
        
        deleteButton.MouseButton1Click:Connect(function()
            self:DeleteTemplate(template.ID)
        end)
    end
    
    -- Edit button (only for owner)
    if template.Creator == Player.Name then
        local editButton = Instance.new("TextButton")
        editButton.Size = UDim2.new(0.2, 0, 0, 30)  -- Taller for mobile
        editButton.Position = UDim2.new(0.4, 0, 0, 85)
        editButton.BackgroundColor3 = Color3.fromRGB(60, 150, 80)
        editButton.BorderSizePixel = 0
        editButton.Text = "Edit"
        editButton.TextColor3 = Color3.fromRGB(255, 255, 255)
        editButton.TextSize = 16  -- Larger for mobile
        editButton.Font = Enum.Font.GothamSemibold
        editButton.Parent = entry
        RoundCorners(editButton, 8)
        
        editButton.MouseButton1Click:Connect(function()
            self.TemplateCreationPanel.Visible = true
            self.TemplateNameInput.Text = template.Name
            self.TemplateDescInput.Text = template.Description
        end)
    end
    
    loadButton.MouseButton1Click:Connect(function()
        self:LoadTemplate(template.ID)
    end)
    
    return entry
end

function DeltaGUIMaker:RefreshTemplateList()
    -- Clear current list
    for _, child in pairs(self.TemplateListContainer:GetChildren()) do
        if child:IsA("Frame") then
            child:Destroy()
        end
    end
    
    -- Filter by search term if any
    local searchTerm = self.TemplateSearchBar.Text:lower()
    local filteredTemplates = {}
    
    for _, template in pairs(self.Templates) do
        if searchTerm == "" or template.Name:lower():find(searchTerm) then
            table.insert(filteredTemplates, template)
        end
    end
    
    -- Sort templates (newest first)
    table.sort(filteredTemplates, function(a, b)
        if a.Creator == Player.Name and b.Creator ~= Player.Name then
            return true
        elseif a.Creator ~= Player.Name and b.Creator == Player.Name then
            return false
        else
            return a.Name < b.Name
        end
    end)
    
    -- Recreate template entries
    for _, template in pairs(filteredTemplates) do
        self:CreateTemplateEntry(template)
    end
    
    -- Show message if no templates found
    if #filteredTemplates == 0 then
        local noTemplatesLabel = Instance.new("TextLabel")
        noTemplatesLabel.Size = UDim2.new(1, -20, 0, 40)
        noTemplatesLabel.Position = UDim2.new(0, 10, 0, 10)
        noTemplatesLabel.BackgroundTransparency = 1
        noTemplatesLabel.Text = searchTerm ~= "" 
            and "No templates found matching '" .. searchTerm .. "'" 
            or "No templates available. Create your first template!"
        noTemplatesLabel.TextColor3 = Color3.fromRGB(180, 180, 180)
        noTemplatesLabel.TextSize = 16
        noTemplatesLabel.Font = Enum.Font.GothamSemibold
        noTemplatesLabel.TextWrapped = true
        noTemplatesLabel.Parent = self.TemplateListContainer
    end
end

function DeltaGUIMaker:SaveTemplatesData()
    -- This would normally save to some external storage
    -- For now, it just keeps the templates in memory during the session
    self:ShowNotification("Templates updated.", Color3.fromRGB(80, 255, 120))
end

function DeltaGUIMaker:LoadTemplatesData()
    -- This would normally load from some external storage
    -- For example purposes, we'll create some sample templates if none exist
    if #self.Templates == 0 then
        table.insert(self.Templates, {
            Name = "Basic Frame",
            Description = "A simple frame template with a title and button",
            Creator = "System",
            CreationDate = os.date("%Y-%m-%d"),
            ID = GenerateID(),
            Elements = {
                {
                    Type = "Frame",
                    Position = UDim2.new(0.2, 0, 0.2, 0),
                    Size = UDim2.new(0.6, 0, 0.6, 0),
                    Color = Color3.fromRGB(60, 60, 60),
                    Name = "MainFrame",
                    CornerRadius = 8,
                    ZIndex = 1,
                    Children = {}
                },
                {
                    Type = "TextLabel",
                    Position = UDim2.new(0.25, 0, 0.3, 0),
                    Size = UDim2.new(0.5, 0, 0.1, 0),
                    Color = Color3.fromRGB(100, 100, 100),
                    TextColor = Color3.fromRGB(255, 255, 255),
                    Text = "Template Title",
                    Font = Enum.Font.GothamSemibold,
                    TextSize = 18,
                    Name = "TitleLabel",
                    CornerRadius = 4,
                    ZIndex = 2,
                    Children = {}
                },
                {
                    Type = "TextButton",
                    Position = UDim2.new(0.35, 0, 0.55, 0),
                    Size = UDim2.new(0.3, 0, 0.1, 0),
                    Color = Color3.fromRGB(60, 120, 200),
                    TextColor = Color3.fromRGB(255, 255, 255),
                    Text = "Button",
                    Font = Enum.Font.GothamSemibold,
                    TextSize = 16,
                    Name = "ActionButton",
                    CornerRadius = 6,
                    ZIndex = 2,
                    Children = {}
                }
            }
        })
        
        table.insert(self.Templates, {
            Name = "Mobile Menu",
            Description = "A mobile-friendly menu template with buttons",
            Creator = "System",
            CreationDate = os.date("%Y-%m-%d"),
            ID = GenerateID(),
            Elements = {
                {
                    Type = "Frame",
                    Position = UDim2.new(0.1, 0, 0.1, 0),
                    Size = UDim2.new(0.8, 0, 0.8, 0),
                    Color = Color3.fromRGB(40, 40, 40),
                    Name = "MobileMenuFrame",
                    CornerRadius = 12,
                    ZIndex = 1,
                    Children = {}
                },
                {
                    Type = "TextLabel",
                    Position = UDim2.new(0.2, 0, 0.15, 0),
                    Size = UDim2.new(0.6, 0, 0.1, 0),
                    Color = Color3.fromRGB(50, 50, 50),
                    TextColor = Color3.fromRGB(255, 255, 255),
                    Text = "Mobile Menu",
                    Font = Enum.Font.GothamBold,
                    TextSize = 24,
                    Name = "MenuTitle",
                    CornerRadius = 6,
                    ZIndex = 2,
                    Children = {}
                },
                {
                    Type = "TextButton",
                    Position = UDim2.new(0.25, 0, 0.35, 0),
                    Size = UDim2.new(0.5, 0, 0.12, 0),
                    Color = Color3.fromRGB(60, 120, 200),
                    TextColor = Color3.fromRGB(255, 255, 255),
                    Text = "Option 1",
                    Font = Enum.Font.GothamSemibold,
                    TextSize = 20,
                    Name = "MenuButton1",
                    CornerRadius = 8,
                    ZIndex = 2,
                    Children = {}
                },
                {
                    Type = "TextButton",
                    Position = UDim2.new(0.25, 0, 0.5, 0),
                    Size = UDim2.new(0.5, 0, 0.12, 0),
                    Color = Color3.fromRGB(60, 150, 80),
                    TextColor = Color3.fromRGB(255, 255, 255),
                    Text = "Option 2",
                    Font = Enum.Font.GothamSemibold,
                    TextSize = 20,
                    Name = "MenuButton2",
                    CornerRadius = 8,
                    ZIndex = 2,
                    Children = {}
                },
                {
                    Type = "TextButton",
                    Position = UDim2.new(0.25, 0, 0.65, 0),
                    Size = UDim2.new(0.5, 0, 0.12, 0),
                    Color = Color3.fromRGB(200, 60, 60),
                    TextColor = Color3.fromRGB(255, 255, 255),
                    Text = "Exit",
                    Font = Enum.Font.GothamSemibold,
                    TextSize = 20,
                    Name = "ExitButton",
                    CornerRadius = 8,
                    ZIndex = 2,
                    Children = {}
                }
            }
        })
    end
end

-- Create dropdown function
local function CreateDropdown(parent, position, size, options, defaultOption)
    local dropdown = Instance.new("Frame")
    dropdown.Name = "Dropdown"
    dropdown.Position = position
    dropdown.Size = size
    dropdown.BackgroundColor3 = Color3.fromRGB(60, 150, 80)
    dropdown.BorderSizePixel = 0
    dropdown.Parent = parent
    RoundCorners(dropdown, 8)
    
    local button = Instance.new("TextButton")
    button.Name = "DropdownButton"
    button.Size = UDim2.new(1, 0, 1, 0)
    button.BackgroundColor3 = Color3.fromRGB(60, 150, 80)
    button.BorderSizePixel = 0
    button.Text = defaultOption or options[1] or "Select..."
    button.TextColor3 = Color3.fromRGB(255, 255, 255)
    button.Font = Enum.Font.GothamSemibold
    button.TextSize = 16  -- Larger for mobile
    button.Parent = dropdown
    RoundCorners(button, 8)
    
    local arrow = Instance.new("TextLabel")
    arrow.Name = "Arrow"
    arrow.Size = UDim2.new(0, 25, 0, 25)  -- Larger for mobile
    arrow.Position = UDim2.new(1, -30, 0.5, -12)
    arrow.BackgroundTransparency = 1
    arrow.Text = "▼"
    arrow.TextColor3 = Color3.fromRGB(255, 255, 255)
    arrow.TextSize = 16  -- Larger for mobile
    arrow.Font = Enum.Font.GothamBold
    arrow.Parent = button
    
    local dropdownList = Instance.new("Frame")
    dropdownList.Name = "OptionsList"
    dropdownList.Size = UDim2.new(1, 0, 0, #options * 35 + 10)  -- Taller items for mobile
    dropdownList.Position = UDim2.new(0, 0, 1, 5)
    dropdownList.BackgroundColor3 = Color3.fromRGB(50, 130, 70)
    dropdownList.BorderSizePixel = 0
    dropdownList.Visible = false
    dropdownList.ZIndex = 10
    dropdownList.Parent = dropdown
    RoundCorners(dropdownList, 8)
    
    local optionsContainer = Instance.new("ScrollingFrame")
    optionsContainer.Name = "OptionsContainer"
    optionsContainer.Size = UDim2.new(1, -10, 1, -10)
    optionsContainer.Position = UDim2.new(0, 5, 0, 5)
    optionsContainer.BackgroundTransparency = 1
    optionsContainer.BorderSizePixel = 0
    optionsContainer.ScrollBarThickness = 6  -- Thicker for mobile
    optionsContainer.ZIndex = 10
    optionsContainer.ScrollingDirection = Enum.ScrollingDirection.Y
    optionsContainer.CanvasSize = UDim2.new(0, 0, 0, #options * 35)  -- Taller items for mobile
    optionsContainer.AutomaticCanvasSize = Enum.AutomaticSize.Y
    optionsContainer.Parent = dropdownList
    
    local optionsList = Instance.new("UIListLayout")
    optionsList.SortOrder = Enum.SortOrder.LayoutOrder
    optionsList.Padding = UDim.new(0, 5)
    optionsList.Parent = optionsContainer
    
    for i, option in ipairs(options) do
        local optionButton = Instance.new("TextButton")
        optionButton.Name = "Option_" .. i
        optionButton.Size = UDim2.new(1, -10, 0, 30)  -- Taller for mobile
        optionButton.BackgroundColor3 = Color3.fromRGB(60, 150, 80)
        optionButton.BorderSizePixel = 0
        optionButton.Text = option
        optionButton.TextColor3 = Color3.fromRGB(255, 255, 255)
        optionButton.Font = Enum.Font.GothamSemibold
        optionButton.TextSize = 16  -- Larger for mobile
        optionButton.ZIndex = 11
        optionButton.Parent = optionsContainer
        RoundCorners(optionButton, 6)
        
        optionButton.MouseButton1Click:Connect(function()
            button.Text = option
            dropdownList.Visible = false
        end)
    end
    
    button.MouseButton1Click:Connect(function()
        dropdownList.Visible = not dropdownList.Visible
        arrow.Text = dropdownList.Visible and "▲" or "▼"
    end)
    
    -- Close dropdown when clicking elsewhere
    UserInputService.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            local position = input.Position
            local dropdownPosition = dropdown.AbsolutePosition
            local dropdownSize = dropdown.AbsoluteSize
            local listPosition = dropdownList.AbsolutePosition
            local listSize = dropdownList.AbsoluteSize
            
            if dropdownList.Visible then
                local inDropdown = position.X >= dropdownPosition.X and position.X <= dropdownPosition.X + dropdownSize.X and
                                   position.Y >= dropdownPosition.Y and position.Y <= dropdownPosition.Y + dropdownSize.Y
                                   
                local inList = position.X >= listPosition.X and position.X <= listPosition.X + listSize.X and
                               position.Y >= listPosition.Y and position.Y <= listPosition.Y + listSize.Y
                               
                if not inDropdown and not inList then
                    dropdownList.Visible = false
                    arrow.Text = "▼"
                end
            end
        end
    end)
    
    return dropdown
end

-- Create toolbar buttons including template system button
function DeltaGUIMaker:CreateToolbarButtons()
    -- Calculate button width based on how many buttons we need
    local buttonCount = 10  -- New, Save, Load, Export, Undo, Redo, Grid, Align, Color, Templates
    local buttonWidth = 1 / buttonCount
    
    -- Mobile-optimized styling
    local buttonHeight = 0.9
    local buttonY = 0.05
    local fontSize = 14
    local cornerRadius = 4
    
    -- New button
    self.NewButton = Instance.new("TextButton")
    self.NewButton.Name = "NewButton"
    self.NewButton.Size = UDim2.new(buttonWidth, 0, buttonHeight, 0)
    self.NewButton.Position = UDim2.new(0, 0, buttonY, 0)
    self.NewButton.BackgroundColor3 = Color3.fromRGB(60, 150, 80)
    self.NewButton.BorderSizePixel = 0
    self.NewButton.Text = "New"
    self.NewButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.NewButton.TextSize = fontSize
    self.NewButton.Font = Enum.Font.GothamSemibold
    self.NewButton.Parent = self.Toolbar
    RoundCorners(self.NewButton, cornerRadius)
    
    -- Save button
    self.SaveButton = Instance.new("TextButton")
    self.SaveButton.Name = "SaveButton"
    self.SaveButton.Size = UDim2.new(buttonWidth, 0, buttonHeight, 0)
    self.SaveButton.Position = UDim2.new(buttonWidth * 1, 0, buttonY, 0)
    self.SaveButton.BackgroundColor3 = Color3.fromRGB(60, 120, 200)
    self.SaveButton.BorderSizePixel = 0
    self.SaveButton.Text = "Save"
    self.SaveButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.SaveButton.TextSize = fontSize
    self.SaveButton.Font = Enum.Font.GothamSemibold
    self.SaveButton.Parent = self.Toolbar
    RoundCorners(self.SaveButton, cornerRadius)
    
    -- Load button
    self.LoadButton = Instance.new("TextButton")
    self.LoadButton.Name = "LoadButton"
    self.LoadButton.Size = UDim2.new(buttonWidth, 0, buttonHeight, 0)
    self.LoadButton.Position = UDim2.new(buttonWidth * 2, 0, buttonY, 0)
    self.LoadButton.BackgroundColor3 = Color3.fromRGB(60, 120, 200)
    self.LoadButton.BorderSizePixel = 0
    self.LoadButton.Text = "Load"
    self.LoadButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.LoadButton.TextSize = fontSize
    self.LoadButton.Font = Enum.Font.GothamSemibold
    self.LoadButton.Parent = self.Toolbar
    RoundCorners(self.LoadButton, cornerRadius)
    
    -- Export button
    self.ExportButton = Instance.new("TextButton")
    self.ExportButton.Name = "ExportButton"
    self.ExportButton.Size = UDim2.new(buttonWidth, 0, buttonHeight, 0)
    self.ExportButton.Position = UDim2.new(buttonWidth * 3, 0, buttonY, 0)
    self.ExportButton.BackgroundColor3 = Color3.fromRGB(150, 100, 200)
    self.ExportButton.BorderSizePixel = 0
    self.ExportButton.Text = "Export"
    self.ExportButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.ExportButton.TextSize = fontSize
    self.ExportButton.Font = Enum.Font.GothamSemibold
    self.ExportButton.Parent = self.Toolbar
    RoundCorners(self.ExportButton, cornerRadius)
    
    -- Undo button
    self.UndoButton = Instance.new("TextButton")
    self.UndoButton.Name = "UndoButton"
    self.UndoButton.Size = UDim2.new(buttonWidth, 0, buttonHeight, 0)
    self.UndoButton.Position = UDim2.new(buttonWidth * 4, 0, buttonY, 0)
    self.UndoButton.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
    self.UndoButton.BorderSizePixel = 0
    self.UndoButton.Text = "Undo"
    self.UndoButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.UndoButton.TextSize = fontSize
    self.UndoButton.Font = Enum.Font.GothamSemibold
    self.UndoButton.Parent = self.Toolbar
    RoundCorners(self.UndoButton, cornerRadius)
    
    -- Redo button
    self.RedoButton = Instance.new("TextButton")
    self.RedoButton.Name = "RedoButton"
    self.RedoButton.Size = UDim2.new(buttonWidth, 0, buttonHeight, 0)
    self.RedoButton.Position = UDim2.new(buttonWidth * 5, 0, buttonY, 0)
    self.RedoButton.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
    self.RedoButton.BorderSizePixel = 0
    self.RedoButton.Text = "Redo"
    self.RedoButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.RedoButton.TextSize = fontSize
    self.RedoButton.Font = Enum.Font.GothamSemibold
    self.RedoButton.Parent = self.Toolbar
    RoundCorners(self.RedoButton, cornerRadius)
    
    -- Grid button
    self.GridButton = Instance.new("TextButton")
    self.GridButton.Name = "GridButton"
    self.GridButton.Size = UDim2.new(buttonWidth, 0, buttonHeight, 0)
    self.GridButton.Position = UDim2.new(buttonWidth * 6, 0, buttonY, 0)
    self.GridButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    self.GridButton.BorderSizePixel = 0
    self.GridButton.Text = "Grid"
    self.GridButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.GridButton.TextSize = fontSize
    self.GridButton.Font = Enum.Font.GothamSemibold
    self.GridButton.Parent = self.Toolbar
    RoundCorners(self.GridButton, cornerRadius)
    
    -- Align button
    self.AlignButton = Instance.new("TextButton")
    self.AlignButton.Name = "AlignButton"
    self.AlignButton.Size = UDim2.new(buttonWidth, 0, buttonHeight, 0)
    self.AlignButton.Position = UDim2.new(buttonWidth * 7, 0, buttonY, 0)
    self.AlignButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    self.AlignButton.BorderSizePixel = 0
    self.AlignButton.Text = "Align"
    self.AlignButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.AlignButton.TextSize = fontSize
    self.AlignButton.Font = Enum.Font.GothamSemibold
    self.AlignButton.Parent = self.Toolbar
    RoundCorners(self.AlignButton, cornerRadius)
    
    -- Color button
    self.ColorButton = Instance.new("TextButton")
    self.ColorButton.Name = "ColorButton"
    self.ColorButton.Size = UDim2.new(buttonWidth, 0, buttonHeight, 0)
    self.ColorButton.Position = UDim2.new(buttonWidth * 8, 0, buttonY, 0)
    self.ColorButton.BackgroundColor3 = Color3.fromRGB(200, 100, 80)
    self.ColorButton.BorderSizePixel = 0
    self.ColorButton.Text = "Color"
    self.ColorButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.ColorButton.TextSize = fontSize
    self.ColorButton.Font = Enum.Font.GothamSemibold
    self.ColorButton.Parent = self.Toolbar
    RoundCorners(self.ColorButton, cornerRadius)
    
    -- Templates button
    self.TemplatesButton = Instance.new("TextButton")
    self.TemplatesButton.Name = "TemplatesButton"
    self.TemplatesButton.Size = UDim2.new(buttonWidth, 0, buttonHeight, 0)
    self.TemplatesButton.Position = UDim2.new(buttonWidth * 9, 0, buttonY, 0)
    self.TemplatesButton.BackgroundColor3 = Color3.fromRGB(200, 150, 50)
    self.TemplatesButton.BorderSizePixel = 0
    self.TemplatesButton.Text = "Templates"
    self.TemplatesButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    self.TemplatesButton.TextSize = fontSize
    self.TemplatesButton.Font = Enum.Font.GothamSemibold
    self.TemplatesButton.Parent = self.Toolbar
    RoundCorners(self.TemplatesButton, cornerRadius)
    
    -- Button events
    self.TemplatesButton.MouseButton1Click:Connect(function()
        self:ShowTemplatePanel()
    end)
    
    -- Button tooltips for mobile (simplified tap-and-hold tooltips)
    self:CreateButtonTooltip(self.TemplatesButton, "Browse and use pre-made templates")
end

-- Show notification function
function DeltaGUIMaker:ShowNotification(message, color)
    -- Create notification frame
    local notification = Instance.new("Frame")
    notification.Name = "Notification"
    notification.Size = UDim2.new(0.5, 0, 0.08, 0)  -- Taller for mobile
    notification.Position = UDim2.new(0.25, 0, 0.9, 0)
    notification.BackgroundColor3 = color or Color3.fromRGB(60, 60, 60)
    notification.BorderSizePixel = 0
    notification.Parent = self.MainUI
    notification.ZIndex = 1000
    RoundCorners(notification, 10)  -- More rounded for mobile
    
    -- Text label
    local text = Instance.new("TextLabel")
    text.Size = UDim2.new(1, 0, 1, 0)
    text.Position = UDim2.new(0, 0, 0, 0)
    text.BackgroundTransparency = 1
    text.Text = message
    text.TextColor3 = Color3.fromRGB(255, 255, 255)
    text.TextSize = 16  -- Larger for mobile
    text.Font = Enum.Font.GothamSemibold
    text.ZIndex = 1001
    text.Parent = notification
    
    -- Fade out animation
    spawn(function()
        wait(0.2)
        local fadeStart = tick()
        local fadeTime = 2.5  -- Show notification for longer on mobile
        
        while tick() - fadeStart < fadeTime do
            local alpha = 1 - ((tick() - fadeStart) / fadeTime)
            notification.BackgroundTransparency = 1 - alpha
            text.TextTransparency = 1 - alpha
            RunService.Heartbeat:Wait()
        end
        
        notification:Destroy()
    end)
end

-- Create button tooltip for mobile
function DeltaGUIMaker:CreateButtonTooltip(button, tooltipText)
    local touchTimer = nil
    local tooltipShown = false
    local tooltipInstance = nil
    
    -- For mobile, show tooltip on longer touch
    button.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then
            -- Start timer to show tooltip after holding
            if touchTimer then touchTimer:Disconnect() end
            
            touchTimer = RunService.Heartbeat:Connect(function()
                if tick() - input.StartTime >= 0.7 and not tooltipShown then
                    -- Create tooltip
                    tooltipInstance = Instance.new("Frame")
                    tooltipInstance.Name = "Tooltip"
                    tooltipInstance.Size = UDim2.new(0, 200, 0, 40)  -- Fixed size for mobile
                    tooltipInstance.Position = UDim2.new(0, input.Position.X, 0, input.Position.Y - 50)  -- Above touch
                    tooltipInstance.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
                    tooltipInstance.BorderSizePixel = 0
                    tooltipInstance.ZIndex = 1000
                    tooltipInstance.Parent = self.MainUI
                    RoundCorners(tooltipInstance, 8)
                    
                    local tooltipTextLabel = Instance.new("TextLabel")
                    tooltipTextLabel.Size = UDim2.new(1, -10, 1, -10)
                    tooltipTextLabel.Position = UDim2.new(0, 5, 0, 5)
                    tooltipTextLabel.BackgroundTransparency = 1
                    tooltipTextLabel.Text = tooltipText
                    tooltipTextLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
                    tooltipTextLabel.TextSize = 14
                    tooltipTextLabel.Font = Enum.Font.Gotham
                    tooltipTextLabel.TextWrapped = true
                    tooltipTextLabel.ZIndex = 1001
                    tooltipTextLabel.Parent = tooltipInstance
                    
                    tooltipShown = true
                    touchTimer:Disconnect()
                    touchTimer = nil
                end
            end)
        end
    end)
    
    button.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then
            if touchTimer then 
                touchTimer:Disconnect()
                touchTimer = nil
            end
            
            if tooltipInstance then
                tooltipInstance:Destroy()
                tooltipInstance = nil
            end
            
            tooltipShown = false
        end
    end)
end


-- Create element buttons for the palette
function DeltaGUIMaker:CreateElementButtons()
    -- Mobile-optimized styling - larger buttons for easy touch
    local buttonHeight = 50  -- Taller for better touch targets
    local padding = 8  -- More space between buttons
    local fontSize = 16  -- Larger text for readability
    local cornerRadius = 8  -- More rounded corners
    
    -- Element button data
    local elementTypes = {
        {
            Type = "Frame", 
            Name = "Frame", 
            Color = Color3.fromRGB(60, 60, 60),
            Description = "Container for other elements"
        },
        {
            Type = "TextLabel", 
            Name = "Text Label", 
            Color = Color3.fromRGB(80, 80, 80),
            Description = "Display text"
        },
        {
            Type = "TextButton", 
            Name = "Button", 
            Color = Color3.fromRGB(60, 120, 200),
            Description = "Clickable button"
        },
        {
            Type = "TextBox", 
            Name = "Text Box", 
            Color = Color3.fromRGB(70, 70, 70),
            Description = "Input text field"
        },
        {
            Type = "ImageLabel", 
            Name = "Image", 
            Color = Color3.fromRGB(80, 180, 120),
            Description = "Display images"
        },
        {
            Type = "ScrollingFrame", 
            Name = "Scroll Frame", 
            Color = Color3.fromRGB(70, 70, 90),
            Description = "Scrollable container"
        },
        {
            Type = "ImageButton", 
            Name = "Image Button", 
            Color = Color3.fromRGB(60, 140, 200),
            Description = "Clickable image"
        }
    }
    
    -- Create buttons
    for i, elementData in ipairs(elementTypes) do
        -- Element button container
        local buttonContainer = Instance.new("Frame")
        buttonContainer.Name = elementData.Type .. "Button"
        buttonContainer.Size = UDim2.new(1, -padding*2, 0, buttonHeight)
        buttonContainer.Position = UDim2.new(0, padding, 0, (i-1) * (buttonHeight + padding))
        buttonContainer.BackgroundColor3 = elementData.Color
        buttonContainer.BorderSizePixel = 0
        buttonContainer.Parent = self.ElementPaletteScroll
        RoundCorners(buttonContainer, cornerRadius)
        
        -- Element name
        local nameLabel = Instance.new("TextLabel")
        nameLabel.Size = UDim2.new(1, 0, 1, 0)
        nameLabel.BackgroundTransparency = 1
        nameLabel.Text = elementData.Name
        nameLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
        nameLabel.TextSize = fontSize
        nameLabel.Font = Enum.Font.GothamSemibold
        nameLabel.Parent = buttonContainer
        
        -- Make the container draggable for mobile
        buttonContainer.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
                -- Start dragging a new element
                local mousePos = UserInputService:GetMouseLocation()
                local viewportSize = Camera.ViewportSize
                
                -- Create a preview element that follows the cursor/touch
                local preview = Instance.new("Frame")
                preview.Size = UDim2.new(0, 100, 0, 100)  -- Default size
                preview.Position = UDim2.new(0, mousePos.X - viewportSize.X/2, 0, mousePos.Y - viewportSize.Y/2)
                preview.BackgroundColor3 = elementData.Color
                preview.BorderSizePixel = 0
                preview.BackgroundTransparency = 0.5  -- Semi-transparent for preview
                preview.Parent = self.MainUI
                RoundCorners(preview, cornerRadius)
                
                -- For specific element types, customize the preview
                if elementData.Type == "TextLabel" or elementData.Type == "TextButton" or elementData.Type == "TextBox" then
                    local previewText = Instance.new("TextLabel")
                    previewText.Size = UDim2.new(1, 0, 1, 0)
                    previewText.BackgroundTransparency = 1
                    previewText.Text = elementData.Type == "TextBox" and "Input..." or elementData.Type == "TextButton" and "Button" or "Text"
                    previewText.TextColor3 = Color3.fromRGB(255, 255, 255)
                    previewText.TextSize = 16
                    previewText.Font = Enum.Font.GothamSemibold
                    previewText.Parent = preview
                elseif elementData.Type == "ImageLabel" or elementData.Type == "ImageButton" then
                    local imageIcon = Instance.new("TextLabel")
                    imageIcon.Size = UDim2.new(1, 0, 1, 0)
                    imageIcon.BackgroundTransparency = 1
                    imageIcon.Text = "🖼️"  -- Image emoji for preview
                    imageIcon.TextColor3 = Color3.fromRGB(255, 255, 255)
                    imageIcon.TextSize = 32  -- Larger for visibility
                    imageIcon.Font = Enum.Font.GothamSemibold
                    imageIcon.Parent = preview
                end
                
                -- Show description tooltip
                local tooltip = Instance.new("Frame")
                tooltip.Size = UDim2.new(0, 200, 0, 40)
                tooltip.Position = UDim2.new(0, mousePos.X - viewportSize.X/2 + 120, 0, mousePos.Y - viewportSize.Y/2 - 50)
                tooltip.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
                tooltip.BorderSizePixel = 0
                tooltip.ZIndex = 1000
                tooltip.Parent = self.MainUI
                RoundCorners(tooltip, 6)
                
                local tooltipText = Instance.new("TextLabel")
                tooltipText.Size = UDim2.new(1, -10, 1, -10)
                tooltipText.Position = UDim2.new(0, 5, 0, 5)
                tooltipText.BackgroundTransparency = 1
                tooltipText.Text = elementData.Description
                tooltipText.TextColor3 = Color3.fromRGB(255, 255, 255)
                tooltipText.TextSize = 14
                tooltipText.Font = Enum.Font.Gotham
                tooltipText.TextWrapped = true
                tooltipText.ZIndex = 1001
                tooltipText.Parent = tooltip
                
                -- Track dragging with mouse/touch movement
                local dragConnection
                dragConnection = UserInputService.InputChanged:Connect(function(inputChanged)
                    if inputChanged.UserInputType == Enum.UserInputType.MouseMovement or inputChanged.UserInputType == Enum.UserInputType.Touch then
                        -- Update preview position
                        local newPos = UserInputService:GetMouseLocation()
                        preview.Position = UDim2.new(0, newPos.X - viewportSize.X/2, 0, newPos.Y - viewportSize.Y/2)
                        
                        -- Update tooltip position
                        tooltip.Position = UDim2.new(0, newPos.X - viewportSize.X/2 + 120, 0, newPos.Y - viewportSize.Y/2 - 50)
                    end
                end)
                
                -- Handle touch/mouse release
                local releaseConnection
                releaseConnection = UserInputService.InputEnded:Connect(function(inputEnded)
                    if inputEnded.UserInputType == Enum.UserInputType.Touch or inputEnded.UserInputType == Enum.UserInputType.MouseButton1 then
                        dragConnection:Disconnect()
                        releaseConnection:Disconnect()
                        
                        -- Determine if the element is being dropped in the work area
                        local finalPos = UserInputService:GetMouseLocation()
                        local workAreaPos = self.WorkArea.AbsolutePosition
                        local workAreaSize = self.WorkArea.AbsoluteSize
                        
                        if finalPos.X >= workAreaPos.X and finalPos.X <= workAreaPos.X + workAreaSize.X and
                           finalPos.Y >= workAreaPos.Y and finalPos.Y <= workAreaPos.Y + workAreaSize.Y then
                            -- Calculate position relative to the grid container
                            local gridPos = self.GridContainer.AbsolutePosition
                            local gridSize = self.GridContainer.AbsoluteSize
                            
                            -- Position in scale relative to grid container
                            local relX = (finalPos.X - gridPos.X) / gridSize.X
                            local relY = (finalPos.Y - gridPos.Y) / gridSize.Y
                            
                            -- Create the actual element
                            self:CreateNewElement(
                                elementData.Type,
                                UDim2.new(relX - 0.05, 0, relY - 0.05, 0),  -- Center element on cursor/touch position
                                UDim2.new(0.1, 0, 0.1, 0),  -- Default size (10% of container)
                                elementData.Color
                            )
                            
                            -- Show touch feedback
                            local feedback = Instance.new("Frame")
                            feedback.Size = UDim2.new(0, 50, 0, 50)
                            feedback.Position = UDim2.new(0, finalPos.X - viewportSize.X/2 - 25, 0, finalPos.Y - viewportSize.Y/2 - 25)
                            feedback.BackgroundColor3 = Color3.fromRGB(80, 255, 120)
                            feedback.BackgroundTransparency = 0.5
                            feedback.BorderSizePixel = 0
                            feedback.ZIndex = 1000
                            feedback.Parent = self.MainUI
                            RoundCorners(feedback, 25)  -- Make it circular
                            
                            -- Fade out feedback
                            spawn(function()
                                for i = 1, 10 do
                                    feedback.BackgroundTransparency = 0.5 + (i * 0.05)
                                    feedback.Size = UDim2.new(0, 50 + i*5, 0, 50 + i*5)
                                    feedback.Position = UDim2.new(0, finalPos.X - viewportSize.X/2 - 25 - i*2.5, 0, finalPos.Y - viewportSize.Y/2 - 25 - i*2.5)
                                    wait(0.02)
                                end
                                feedback:Destroy()
                            end)
                        end
                        
                        preview:Destroy()
                        tooltip:Destroy()
                    end
                end)
            end
        end)
        
        -- Add tooltip for element description (on click for mobile)
        self:CreateButtonTooltip(buttonContainer, elementData.Description)
    end
end

-- Create a new element
function DeltaGUIMaker:CreateNewElement(elementType, position, size, color)
    -- Generate unique identifier
    local elementId = GenerateID()
    
    -- Default properties
    local elementName = elementType .. elementId:sub(1, 4)
    color = color or self.CurrentColor
    
    -- Create element data structure
    local elementData = {
        ID = elementId,
        Type = elementType,
        Position = position,
        Size = size,
        Color = color,
        Name = elementName,
        ZIndex = 1,
        CornerRadius = 8,  -- Mobile-friendly rounded corners
        Children = {}
    }
    
    -- Add type-specific properties
    if elementType == "TextLabel" or elementType == "TextButton" or elementType == "TextBox" then
        elementData.Text = elementType == "TextBox" and "Input..." or elementType == "TextButton" and "Button" or "Text"
        elementData.TextColor = Color3.fromRGB(255, 255, 255)
        elementData.Font = Enum.Font.GothamSemibold
        elementData.TextSize = 16  -- Larger text for mobile
    elseif elementType == "ImageLabel" or elementType == "ImageButton" then
        elementData.Image = ""
        elementData.ScaleType = Enum.ScaleType.Slice
        elementData.SliceCenter = Rect.new(0, 0, 0, 0)
    elseif elementType == "ScrollingFrame" then
        elementData.CanvasSize = UDim2.new(0, 0, 2, 0)
        elementData.ScrollBarThickness = 6  -- Thicker for mobile
    end
    
    -- Create the instance
    elementData.Instance = self:CreateElementInstance(elementData)
    
    -- Save to undo stack
    table.insert(self.UndoStack, {
        Action = "Create",
        Element = DeepCopy(elementData)
    })
    self.RedoStack = {}  -- Clear redo stack
    
    -- Add to elements table
    table.insert(self.Elements, elementData)
    
    -- Select the new element
    self:SelectElement(elementData)
    
    return elementData
end

-- Create instance of an element from data
function DeltaGUIMaker:CreateElementInstance(elementData)
    -- Create the instance
    local instance = Instance.new(elementData.Type)
    instance.Name = elementData.Name
    instance.Position = elementData.Position
    instance.Size = elementData.Size
    instance.BackgroundColor3 = elementData.Color
    instance.ZIndex = elementData.ZIndex or 1
    instance.BorderSizePixel = 0  -- No border for clean look
    instance.Parent = self.GridContainer
    
    -- Round corners for mobile-friendly appearance
    RoundCorners(instance, elementData.CornerRadius or 8)
    
    -- Add type-specific properties
    if elementData.Type == "TextLabel" or elementData.Type == "TextButton" or elementData.Type == "TextBox" then
        instance.Text = elementData.Text or ""
        instance.TextColor3 = elementData.TextColor or Color3.fromRGB(255, 255, 255)
        instance.Font = elementData.Font or Enum.Font.GothamSemibold
        instance.TextSize = elementData.TextSize or 16  -- Larger for mobile
        
        -- For TextBox, add clear text on focus behavior
        if elementData.Type == "TextBox" then
            instance.ClearTextOnFocus = elementData.ClearTextOnFocus or false
            instance.PlaceholderText = elementData.PlaceholderText or "Enter text..."
            instance.PlaceholderColor3 = elementData.PlaceholderColor3 or Color3.fromRGB(180, 180, 180)
        end
    elseif elementData.Type == "ImageLabel" or elementData.Type == "ImageButton" then
        instance.Image = elementData.Image or ""
        instance.ScaleType = elementData.ScaleType or Enum.ScaleType.Stretch
        instance.SliceCenter = elementData.SliceCenter or Rect.new(0, 0, 0, 0)
        instance.ImageTransparency = elementData.ImageTransparency or 0
    elseif elementData.Type == "ScrollingFrame" then
        instance.CanvasSize = elementData.CanvasSize or UDim2.new(0, 0, 2, 0)
        instance.ScrollBarThickness = elementData.ScrollBarThickness or 6  -- Thicker for mobile
        instance.ScrollingDirection = elementData.ScrollingDirection or Enum.ScrollingDirection.Y
        instance.AutomaticCanvasSize = elementData.AutomaticCanvasSize or Enum.AutomaticSize.Y
    end
    
    -- Add shadow for depth
    ApplyShadow(instance, 2, 0.7)
    
    -- Make element selectable and draggable
    instance.SelectionImageObject = Instance.new("ImageLabel")
    instance.Active = true
    
    -- Touch controls for mobile
    instance.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            -- Select the element
            for i, element in pairs(self.Elements) do
                if element.Instance == instance then
                    self:SelectElement(element)
                    
                    -- Start dragging on mobile (after a slight delay to distinguish tap from drag)
                    spawn(function()
                        local startTime = tick()
                        local startPos = input.Position
                        local hasMoved = false
                        
                        while UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1) or 
                              UserInputService:IsMouseButtonPressed(Enum.UserInputType.Touch) do
                            local currentPos = UserInputService:GetMouseLocation()
                            local delta = (currentPos - startPos).Magnitude
                            
                            if delta > 10 then  -- Movement threshold for mobile
                                hasMoved = true
                                break
                            end
                            
                            if tick() - startTime > 0.15 then  -- Short delay for tap vs drag
                                break
                            end
                            
                            wait()
                        end
                        
                        -- If moved enough, start drag operation
                        if hasMoved then
                            self.DraggingElement = element
                            self.DraggingType = "move"
                        end
                    end)
                    break
                end
            end
        end
    end)
    
    return instance
end


-- Register all UI events
function DeltaGUIMaker:RegisterEvents()
    -- Template panel events
    self.TemplateSearchButton.MouseButton1Click:Connect(function()
        self:RefreshTemplateList()
    end)
    
    self.TemplateSearchBar.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            self:RefreshTemplateList()
        end
    end)
    
    self.CloseTemplateButton.MouseButton1Click:Connect(function()
        self:HideTemplatePanel()
    end)
    
    self.CreateTemplateButton.MouseButton1Click:Connect(function()
        self:ShowTemplateCreationPanel()
    end)
    
    -- Template creation panel events
    self.SaveTemplateButton.MouseButton1Click:Connect(function()
        self:SaveTemplate()
    end)
    
    self.CancelTemplateButton.MouseButton1Click:Connect(function()
        self:HideTemplateCreationPanel()
    end)
    
    -- Other button events
    self.CloseButton.MouseButton1Click:Connect(function()
        self:Close()
    end)
    
    -- Touch input for mobile
    UserInputService.TouchTap:Connect(function(touchPositions, gameProcessedEvent)
        if gameProcessedEvent then return end
        
        -- Handle touch selection and interaction
        -- This is a simplified version of touch handling
        if #touchPositions > 0 then
            local touchPosition = touchPositions[1]
            -- Process touch input for element selection/manipulation
        end
    end)
    
    -- Input began event for tracking element manipulation
    UserInputService.InputBegan:Connect(function(input, gameProcessedEvent)
        if gameProcessedEvent then return end
        
        -- Handle input began
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            -- Process touch/click input
        elseif input.UserInputType == Enum.UserInputType.Keyboard then
            -- Handle keyboard shortcuts (for non-mobile use cases)
            if input.KeyCode == Enum.KeyCode.Delete and self.SelectedElement then
                self:DeleteSelectedElement()
            elseif input.KeyCode == Enum.KeyCode.Z and UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
                self:Undo()
            elseif input.KeyCode == Enum.KeyCode.Y and UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
                self:Redo()
            end
        end
    end)
    
    -- Input changed event for tracking element manipulation
    UserInputService.InputChanged:Connect(function(input, gameProcessedEvent)
        if gameProcessedEvent then return end
        
        -- Handle drag operations for mobile
        if (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) and self.DraggingElement then
            if self.DraggingType == "move" then
                -- Move the element
                local mousePos = UserInputService:GetMouseLocation()
                local viewportSize = Camera.ViewportSize
                local gridPos = self.GridContainer.AbsolutePosition
                local gridSize = self.GridContainer.AbsoluteSize
                
                -- Calculate position within grid
                local relX = (mousePos.X - gridPos.X) / gridSize.X
                local relY = (mousePos.Y - gridPos.Y) / gridSize.Y
                
                -- Snap to grid if enabled
                if self.SnapToGrid then
                    local gridSizeX = self.GridSize / gridSize.X
                    local gridSizeY = self.GridSize / gridSize.Y
                    relX = math.floor(relX / gridSizeX) * gridSizeX
                    relY = math.floor(relY / gridSizeY) * gridSizeY
                end
                
                -- Update the element's position
                local halfSizeX = self.DraggingElement.Size.X.Scale / 2
                local halfSizeY = self.DraggingElement.Size.Y.Scale / 2
                
                -- Keep within the grid boundaries
                relX = math.max(0, math.min(1 - self.DraggingElement.Size.X.Scale, relX))
                relY = math.max(0, math.min(1 - self.DraggingElement.Size.Y.Scale, relY))
                
                self.DraggingElement.Position = UDim2.new(relX, 0, relY, 0)
                self.DraggingElement.Instance.Position = UDim2.new(relX, 0, relY, 0)
                
                -- Update properties panel
                self:UpdatePropertiesPanel()
            elseif self.DraggingType == "resize" and self.ResizeHandle then
                -- Resize the element
                local mousePos = UserInputService:GetMouseLocation()
                local viewportSize = Camera.ViewportSize
                local gridPos = self.GridContainer.AbsolutePosition
                local gridSize = self.GridContainer.AbsoluteSize
                
                -- Calculate new size
                local startPos = self.ResizeStartPos
                local deltaX = (mousePos.X - startPos.X) / gridSize.X
                local deltaY = (mousePos.Y - startPos.Y) / gridSize.Y
                
                local newSizeX = self.ResizeStartSize.X.Scale + deltaX
                local newSizeY = self.ResizeStartSize.Y.Scale + deltaY
                
                -- Enforce minimum size for mobile touch
                newSizeX = math.max(0.05, newSizeX)  -- Minimum 5% of grid size
                newSizeY = math.max(0.05, newSizeY)  -- Minimum 5% of grid size
                
                -- Snap to grid if enabled
                if self.SnapToGrid then
                    local gridSizeX = self.GridSize / gridSize.X
                    local gridSizeY = self.GridSize / gridSize.Y
                    newSizeX = math.floor(newSizeX / gridSizeX) * gridSizeX
                    newSizeY = math.floor(newSizeY / gridSizeY) * gridSizeY
                end
                
                -- Keep within the grid boundaries
                local posX = self.SelectedElement.Position.X.Scale
                local posY = self.SelectedElement.Position.Y.Scale
                newSizeX = math.min(1 - posX, newSizeX)
                newSizeY = math.min(1 - posY, newSizeY)
                
                -- Update the element's size
                self.SelectedElement.Size = UDim2.new(newSizeX, 0, newSizeY, 0)
                self.SelectedElement.Instance.Size = UDim2.new(newSizeX, 0, newSizeY, 0)
                
                -- Update properties panel
                self:UpdatePropertiesPanel()
            end
        end
    end)
    
    -- Input ended event for finalizing element manipulation
    UserInputService.InputEnded:Connect(function(input, gameProcessedEvent)
        if gameProcessedEvent then return end
        
        -- Finalize drag operations
        if (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1) and self.DraggingElement then
            -- Record the change for undo
            if self.DraggingType == "move" then
                table.insert(self.UndoStack, {
                    Action = "Move",
                    Element = self.DraggingElement,
                    OldPosition = self.DraggingStartPosition,
                    NewPosition = self.DraggingElement.Position
                })
            elseif self.DraggingType == "resize" and self.ResizeHandle and self.SelectedElement then
                table.insert(self.UndoStack, {
                    Action = "Resize",
                    Element = self.SelectedElement,
                    OldSize = self.ResizeStartSize,
                    NewSize = self.SelectedElement.Size
                })
            end
            
            self.RedoStack = {}  -- Clear redo stack
            self.DraggingElement = nil
            self.DraggingType = nil
            self.ResizeHandle = nil
        end
    end)
    
    -- Render loop for grid lines (if enabled)
    RunService.RenderStepped:Connect(function()
        -- Update grid visualization if needed
    end)
    
    -- Load templates on startup
    self:LoadTemplatesData()
end

-- Select an element
function DeltaGUIMaker:SelectElement(element)
    -- Deselect previous element if any
    if self.SelectedElement then
        -- Remove selection highlight
        for _, child in pairs(self.GridContainer:GetChildren()) do
            if child.Name == "SelectionBox" then
                child:Destroy()
            end
        end
    end
    
    -- Set the new selected element
    self.SelectedElement = element
    
    -- Create selection highlight for mobile
    local selectionBox = Instance.new("Frame")
    selectionBox.Name = "SelectionBox"
    selectionBox.Size = UDim2.new(
        element.Size.X.Scale + 0.01, 
        element.Size.X.Offset + 10, 
        element.Size.Y.Scale + 0.01, 
        element.Size.Y.Offset + 10
    )
    selectionBox.Position = UDim2.new(
        element.Position.X.Scale - 0.005, 
        element.Position.X.Offset - 5, 
        element.Position.Y.Scale - 0.005, 
        element.Position.Y.Offset - 5
    )
    selectionBox.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    selectionBox.BackgroundTransparency = 0.7
    selectionBox.BorderSizePixel = 0
    selectionBox.ZIndex = element.ZIndex - 1
    selectionBox.Parent = self.GridContainer
    RoundCorners(selectionBox, element.CornerRadius + 4)
    
    -- Create resize handle (larger for mobile)
    local resizeHandle = Instance.new("TextButton")
    resizeHandle.Name = "ResizeHandle"
    resizeHandle.Size = UDim2.new(0, 30, 0, 30)  -- Larger for mobile
    resizeHandle.Position = UDim2.new(1, -15, 1, -15)
    resizeHandle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    resizeHandle.BorderSizePixel = 0
    resizeHandle.Text = ""
    resizeHandle.ZIndex = element.ZIndex + 1
    resizeHandle.Parent = selectionBox
    RoundCorners(resizeHandle, 15)  -- Make it circular
    
    -- Resize diagonal lines
    local line1 = Instance.new("Frame")
    line1.Name = "Line1"
    line1.Size = UDim2.new(0, 2, 0, 14)
    line1.Position = UDim2.new(0.5, -5, 0.5, -7)
    line1.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    line1.BorderSizePixel = 0
    line1.Rotation = 45
    line1.ZIndex = element.ZIndex + 2
    line1.Parent = resizeHandle
    
    local line2 = Instance.new("Frame")
    line2.Name = "Line2"
    line2.Size = UDim2.new(0, 2, 0, 14)
    line2.Position = UDim2.new(0.5, 5, 0.5, -7)
    line2.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    line2.BorderSizePixel = 0
    line2.Rotation = -45
    line2.ZIndex = element.ZIndex + 2
    line2.Parent = resizeHandle
    
    -- Resize handle events for mobile
    resizeHandle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            self.DraggingElement = element
            self.DraggingType = "resize"
            self.ResizeHandle = resizeHandle
            self.ResizeStartSize = element.Size
            self.ResizeStartPos = input.Position
        end
    end)
    
    -- Update properties panel
    self:UpdatePropertiesPanel()
end

-- Update properties panel with selected element's properties
function DeltaGUIMaker:UpdatePropertiesPanel()
    -- Clear current properties
    for _, child in pairs(self.PropertiesScroll:GetChildren()) do
        if not child:IsA("UIListLayout") then
            child:Destroy()
        end
    end
    
    -- No element selected
    if not self.SelectedElement then
        local noSelectLabel = Instance.new("TextLabel")
        noSelectLabel.Size = UDim2.new(1, -10, 0, 30)
        noSelectLabel.BackgroundTransparency = 1
        noSelectLabel.Text = "No element selected"
        noSelectLabel.TextColor3 = Color3.fromRGB(180, 180, 180)
        noSelectLabel.TextSize = 16  -- Larger for mobile
        noSelectLabel.Font = Enum.Font.Gotham
        noSelectLabel.Parent = self.PropertiesScroll
        return
    end
    
    -- Element info header
    local elementHeader = Instance.new("TextLabel")
    elementHeader.Size = UDim2.new(1, -10, 0, 30)  -- Taller for mobile
    elementHeader.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    elementHeader.BorderSizePixel = 0
    elementHeader.Text = self.SelectedElement.Type .. ": " .. self.SelectedElement.Name
    elementHeader.TextColor3 = Color3.fromRGB(255, 255, 255)
    elementHeader.TextSize = 16  -- Larger for mobile
    elementHeader.Font = Enum.Font.GothamSemibold
    elementHeader.Parent = self.PropertiesScroll
    RoundCorners(elementHeader, 6)
    
    -- Create mobile-friendly property editors
    self:CreatePropertyEditor("Name", self.SelectedElement.Name, "TextBox", function(value)
        self.SelectedElement.Name = value
        self.SelectedElement.Instance.Name = value
        elementHeader.Text = self.SelectedElement.Type .. ": " .. value
    })
    
    -- Position X
    self:CreatePropertyEditor("Position X", string.format("%.3f", self.SelectedElement.Position.X.Scale), "TextBox", function(value)
        local numValue = tonumber(value)
        if numValue then
            numValue = math.max(0, math.min(1, numValue))  -- Clamp between 0 and 1
            self.SelectedElement.Position = UDim2.new(numValue, 0, self.SelectedElement.Position.Y.Scale, 0)
            self.SelectedElement.Instance.Position = self.SelectedElement.Position
        end
    }, "number")
    
    -- Position Y
    self:CreatePropertyEditor("Position Y", string.format("%.3f", self.SelectedElement.Position.Y.Scale), "TextBox", function(value)
        local numValue = tonumber(value)
        if numValue then
            numValue = math.max(0, math.min(1, numValue))  -- Clamp between 0 and 1
            self.SelectedElement.Position = UDim2.new(self.SelectedElement.Position.X.Scale, 0, numValue, 0)
            self.SelectedElement.Instance.Position = self.SelectedElement.Position
        end
    }, "number")
    
    -- Size X
    self:CreatePropertyEditor("Size X", string.format("%.3f", self.SelectedElement.Size.X.Scale), "TextBox", function(value)
        local numValue = tonumber(value)
        if numValue then
            -- Ensure minimum size and within bounds
            numValue = math.max(0.01, math.min(1 - self.SelectedElement.Position.X.Scale, numValue))
            self.SelectedElement.Size = UDim2.new(numValue, 0, self.SelectedElement.Size.Y.Scale, 0)
            self.SelectedElement.Instance.Size = self.SelectedElement.Size
        end
    }, "number")
    
    -- Size Y
    self:CreatePropertyEditor("Size Y", string.format("%.3f", self.SelectedElement.Size.Y.Scale), "TextBox", function(value)
        local numValue = tonumber(value)
        if numValue then
            -- Ensure minimum size and within bounds
            numValue = math.max(0.01, math.min(1 - self.SelectedElement.Position.Y.Scale, numValue))
            self.SelectedElement.Size = UDim2.new(self.SelectedElement.Size.X.Scale, 0, numValue, 0)
            self.SelectedElement.Instance.Size = self.SelectedElement.Size
        end
    }, "number")
    
    -- Color
    local r, g, b = 
        math.floor(self.SelectedElement.Color.R * 255),
        math.floor(self.SelectedElement.Color.G * 255),
        math.floor(self.SelectedElement.Color.B * 255)
    
    self:CreatePropertyEditor("Color", string.format("%d, %d, %d", r, g, b), "Color", function(r, g, b)
        local newColor = Color3.fromRGB(r, g, b)
        self.SelectedElement.Color = newColor
        self.SelectedElement.Instance.BackgroundColor3 = newColor
    })
    
    -- Corner radius
    self:CreatePropertyEditor("Corner Radius", tostring(self.SelectedElement.CornerRadius or 8), "TextBox", function(value)
        local numValue = tonumber(value)
        if numValue then
            numValue = math.max(0, math.min(50, numValue))  -- Clamp between 0 and 50
            self.SelectedElement.CornerRadius = numValue
            
            -- Update corner radius
            for _, child in pairs(self.SelectedElement.Instance:GetChildren()) do
                if child:IsA("UICorner") then
                    child.CornerRadius = UDim.new(0, numValue)
                    break
                end
            end
        end
    }, "number")
    
    -- Z-Index
    self:CreatePropertyEditor("Z-Index", tostring(self.SelectedElement.ZIndex or 1), "TextBox", function(value)
        local numValue = tonumber(value)
        if numValue then
            numValue = math.max(1, math.min(10, numValue))  -- Clamp between 1 and 10
            self.SelectedElement.ZIndex = numValue
            self.SelectedElement.Instance.ZIndex = numValue
        end
    }, "number")
    
    -- Type-specific properties
    if self.SelectedElement.Type == "TextLabel" or self.SelectedElement.Type == "TextButton" or self.SelectedElement.Type == "TextBox" then
        -- Text
        self:CreatePropertyEditor("Text", self.SelectedElement.Text or "", "TextBox", function(value)
            self.SelectedElement.Text = value
            self.SelectedElement.Instance.Text = value
        })
        
        -- Text color
        local tr, tg, tb = 
            math.floor((self.SelectedElement.TextColor or Color3.fromRGB(255, 255, 255)).R * 255),
            math.floor((self.SelectedElement.TextColor or Color3.fromRGB(255, 255, 255)).G * 255),
            math.floor((self.SelectedElement.TextColor or Color3.fromRGB(255, 255, 255)).B * 255)
        
        self:CreatePropertyEditor("Text Color", string.format("%d, %d, %d", tr, tg, tb), "Color", function(r, g, b)
            local newColor = Color3.fromRGB(r, g, b)
            self.SelectedElement.TextColor = newColor
            self.SelectedElement.Instance.TextColor3 = newColor
        })
        
        -- Text size
        self:CreatePropertyEditor("Text Size", tostring(self.SelectedElement.TextSize or 16), "TextBox", function(value)
            local numValue = tonumber(value)
            if numValue then
                numValue = math.max(8, math.min(72, numValue))  -- Clamp between 8 and 72
                self.SelectedElement.TextSize = numValue
                self.SelectedElement.Instance.TextSize = numValue
            end
        }, "number")
        
        -- Font (dropdown)
        local fonts = {"GothamSemibold", "Gotham", "GothamBold", "SourceSans", "SourceSansBold", "Arial", "ArialBold"}
        self:CreatePropertyEditor("Font", (self.SelectedElement.Font and self.SelectedElement.Font.Name) or "GothamSemibold", "Dropdown", function(value)
            local fontEnum = Enum.Font[value]
            if fontEnum then
                self.SelectedElement.Font = fontEnum
                self.SelectedElement.Instance.Font = fontEnum
            end
        }, nil, fonts)
        
        -- Text wrapping (checkbox)
        self:CreatePropertyEditor("Text Wrapped", self.SelectedElement.TextWrapped and "true" or "false", "Checkbox", function(value)
            self.SelectedElement.TextWrapped = value
            self.SelectedElement.Instance.TextWrapped = value
        })
        
        -- Add script button for TextButton
        if self.SelectedElement.Type == "TextButton" then
            self:CreatePropertyEditor("Edit Script", "Open Script Editor", "Button", function()
                self:OpenScriptEditor()
            })
        end
    elseif self.SelectedElement.Type == "ImageLabel" or self.SelectedElement.Type == "ImageButton" then
        -- Image ID
        self:CreatePropertyEditor("Image ID", self.SelectedElement.Image or "", "TextBox", function(value)
            self.SelectedElement.Image = value
            self.SelectedElement.Instance.Image = value
        })
        
        -- Image transparency
        self:CreatePropertyEditor("Transparency", tostring(self.SelectedElement.ImageTransparency or 0), "TextBox", function(value)
            local numValue = tonumber(value)
            if numValue then
                numValue = math.max(0, math.min(1, numValue))  -- Clamp between 0 and 1
                self.SelectedElement.ImageTransparency = numValue
                self.SelectedElement.Instance.ImageTransparency = numValue
            end
        }, "number")
        
        -- Scale type (dropdown)
        local scaleTypes = {"Stretch", "Slice", "Tile", "Fit", "Crop"}
        self:CreatePropertyEditor("Scale Type", (self.SelectedElement.ScaleType and self.SelectedElement.ScaleType.Name) or "Stretch", "Dropdown", function(value)
            local scaleEnum = Enum.ScaleType[value]
            if scaleEnum then
                self.SelectedElement.ScaleType = scaleEnum
                self.SelectedElement.Instance.ScaleType = scaleEnum
            end
        }, nil, scaleTypes)
        
        -- Add script button for ImageButton
        if self.SelectedElement.Type == "ImageButton" then
            self:CreatePropertyEditor("Edit Script", "Open Script Editor", "Button", function()
                self:OpenScriptEditor()
            })
        end
    end
    
    -- Delete button
    local deleteButton = Instance.new("TextButton")
    deleteButton.Size = UDim2.new(1, -20, 0, 40)  -- Taller for mobile
    deleteButton.Position = UDim2.new(0, 10, 0, 0)
    deleteButton.BackgroundColor3 = Color3.fromRGB(200, 60, 60)
    deleteButton.BorderSizePixel = 0
    deleteButton.Text = "Delete Element"
    deleteButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    deleteButton.TextSize = 16  -- Larger for mobile
    deleteButton.Font = Enum.Font.GothamSemibold
    deleteButton.Parent = self.PropertiesScroll
    RoundCorners(deleteButton, 8)
    
    deleteButton.MouseButton1Click:Connect(function()
        self:DeleteSelectedElement()
    end)
end

-- Create a property editor UI element
function DeltaGUIMaker:CreatePropertyEditor(propertyName, value, editorType, callback, validationType, options)
    -- Mobile-optimized styling
    local editorHeight = 40  -- Taller for mobile
    local fontSize = 16  -- Larger text for mobile
    local padding = 10
    
    -- Container frame
    local container = Instance.new("Frame")
    container.Size = UDim2.new(1, -20, 0, editorHeight + 30)  -- Taller for mobile
    container.Position = UDim2.new(0, 10, 0, 0)
    container.BackgroundTransparency = 1
    container.Parent = self.PropertiesScroll
    
    -- Property name label
    local nameLabel = Instance.new("TextLabel")
    nameLabel.Size = UDim2.new(1, 0, 0, 24)  -- Taller for mobile
    nameLabel.Position = UDim2.new(0, 0, 0, 0)
    nameLabel.BackgroundTransparency = 1
    nameLabel.Text = propertyName
    nameLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    nameLabel.TextSize = fontSize
    nameLabel.Font = Enum.Font.Gotham
    nameLabel.TextXAlignment = Enum.TextXAlignment.Left
    nameLabel.Parent = container
    
    -- Different editor types
    if editorType == "TextBox" then
        local editor = Instance.new("TextBox")
        editor.Size = UDim2.new(1, 0, 0, editorHeight)
        editor.Position = UDim2.new(0, 0, 0, 24)
        editor.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        editor.BorderSizePixel = 0
        editor.Text = value
        editor.PlaceholderText = "Enter " .. propertyName .. "..."
        editor.TextColor3 = Color3.fromRGB(255, 255, 255)
        editor.PlaceholderColor3 = Color3.fromRGB(180, 180, 180)
        editor.TextSize = fontSize
        editor.Font = Enum.Font.Gotham
        editor.ClearTextOnFocus = false
        editor.Parent = container
        RoundCorners(editor, 8)
        
        editor.FocusLost:Connect(function(enterPressed)
            local newValue = editor.Text
            
            -- Validate based on type
            if validationType == "number" then
                local numValue = tonumber(newValue)
                if numValue then
                    callback(newValue)
                else
                    editor.Text = value  -- Revert if invalid
                    self:ShowNotification("Please enter a valid number", Color3.fromRGB(255, 80, 80))
                end
            else
                callback(newValue)
            end
        end)
    elseif editorType == "Color" then
        -- Parse RGB values
        local r, g, b = 255, 255, 255
        if value:match("^%d+,%s*%d+,%s*%d+$") then
            r, g, b = value:match("(%d+),%s*(%d+),%s*(%d+)")
            r, g, b = tonumber(r), tonumber(g), tonumber(b)
        end
        
        -- Color preview
        local colorPreview = Instance.new("Frame")
        colorPreview.Size = UDim2.new(0, editorHeight, 0, editorHeight)
        colorPreview.Position = UDim2.new(0, 0, 0, 24)
        colorPreview.BackgroundColor3 = Color3.fromRGB(r, g, b)
        colorPreview.BorderSizePixel = 0
        colorPreview.Parent = container
        RoundCorners(colorPreview, 8)
        
        -- R input
        local redInput = Instance.new("TextBox")
        redInput.Size = UDim2.new(0.3, -padding, 0, editorHeight)
        redInput.Position = UDim2.new(0, editorHeight + padding, 0, 24)
        redInput.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        redInput.BorderSizePixel = 0
        redInput.Text = tostring(r)
        redInput.PlaceholderText = "R"
        redInput.TextColor3 = Color3.fromRGB(255, 255, 255)
        redInput.TextSize = fontSize
        redInput.Font = Enum.Font.Gotham
        redInput.Parent = container
        RoundCorners(redInput, 8)
        
        -- G input
        local greenInput = Instance.new("TextBox")
        greenInput.Size = UDim2.new(0.3, -padding, 0, editorHeight)
        greenInput.Position = UDim2.new(0.35, 0, 0, 24)
        greenInput.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        greenInput.BorderSizePixel = 0
        greenInput.Text = tostring(g)
        greenInput.PlaceholderText = "G"
        greenInput.TextColor3 = Color3.fromRGB(255, 255, 255)
        greenInput.TextSize = fontSize
        greenInput.Font = Enum.Font.Gotham
        greenInput.Parent = container
        RoundCorners(greenInput, 8)
        
        -- B input
        local blueInput = Instance.new("TextBox")
        blueInput.Size = UDim2.new(0.3, -padding, 0, editorHeight)
        blueInput.Position = UDim2.new(0.7, 0, 0, 24)
        blueInput.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        blueInput.BorderSizePixel = 0
        blueInput.Text = tostring(b)
        blueInput.PlaceholderText = "B"
        blueInput.TextColor3 = Color3.fromRGB(255, 255, 255)
        blueInput.TextSize = fontSize
        blueInput.Font = Enum.Font.Gotham
        blueInput.Parent = container
        RoundCorners(blueInput, 8)
        
        -- Update color function
        local function updateColor()
            local newR = tonumber(redInput.Text) or r
            local newG = tonumber(greenInput.Text) or g
            local newB = tonumber(blueInput.Text) or b
            
            -- Clamp RGB values
            newR = math.max(0, math.min(255, newR))
            newG = math.max(0, math.min(255, newG))
            newB = math.max(0, math.min(255, newB))
            
            -- Update preview
            colorPreview.BackgroundColor3 = Color3.fromRGB(newR, newG, newB)
            
            -- Call callback
            callback(newR, newG, newB)
        end
        
        -- Connect events
        redInput.FocusLost:Connect(function() updateColor() end)
        greenInput.FocusLost:Connect(function() updateColor() end)
        blueInput.FocusLost:Connect(function() updateColor() end)
    elseif editorType == "Dropdown" then
        -- Create dropdown using the utility function
        local dropdown = CreateDropdown(container, UDim2.new(0, 0, 0, 24), UDim2.new(1, 0, 0, editorHeight), options, value)
        
        -- Connect event
        dropdown.DropdownButton.MouseButton1Click:Connect(function()
            local optionsList = dropdown.OptionsList
            
            if optionsList.Visible then
                for _, option in pairs(optionsList.OptionsContainer:GetChildren()) do
                    if option:IsA("TextButton") then
                        option.MouseButton1Click:Connect(function()
                            callback(option.Text)
                        end)
                    end
                end
            end
        end)
    elseif editorType == "Checkbox" then
        -- Boolean value
        local isChecked = (value == "true")
        
        -- Checkbox container (bigger for mobile)
        local checkboxContainer = Instance.new("Frame")
        checkboxContainer.Size = UDim2.new(0, editorHeight, 0, editorHeight)
        checkboxContainer.Position = UDim2.new(0, 0, 0, 24)
        checkboxContainer.BackgroundColor3 = isChecked and Color3.fromRGB(60, 150, 80) or Color3.fromRGB(80, 80, 80)
        checkboxContainer.BorderSizePixel = 0
        checkboxContainer.Parent = container
        RoundCorners(checkboxContainer, 8)
        
        -- Checkmark
        local checkmark = Instance.new("TextLabel")
        checkmark.Size = UDim2.new(1, 0, 1, 0)
        checkmark.BackgroundTransparency = 1
        checkmark.Text = isChecked and "✓" or ""
        checkmark.TextColor3 = Color3.fromRGB(255, 255, 255)
        checkmark.TextSize = 24  -- Larger for mobile
        checkmark.Font = Enum.Font.GothamBold
        checkmark.Parent = checkboxContainer
        
        -- Toggle function
        local function toggleCheckbox()
            isChecked = not isChecked
            checkboxContainer.BackgroundColor3 = isChecked and Color3.fromRGB(60, 150, 80) or Color3.fromRGB(80, 80, 80)
            checkmark.Text = isChecked and "✓" or ""
            callback(isChecked)
        end
        
        -- Connect event (for mobile)
        checkboxContainer.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
                toggleCheckbox()
            end
        end)
    elseif editorType == "Button" then
        -- Action button
        local button = Instance.new("TextButton")
        button.Size = UDim2.new(1, 0, 0, editorHeight)
        button.Position = UDim2.new(0, 0, 0, 24)
        button.BackgroundColor3 = Color3.fromRGB(60, 120, 200)
        button.BorderSizePixel = 0
        button.Text = value
        button.TextColor3 = Color3.fromRGB(255, 255, 255)
        button.TextSize = fontSize
        button.Font = Enum.Font.GothamSemibold
        button.Parent = container
        RoundCorners(button, 8)
        
        -- Connect event
        button.MouseButton1Click:Connect(callback)
    end
    
    return container
end

-- Generate code for current GUI
function DeltaGUIMaker:GenerateCode()
    local code = "--[[\n    Generated with Delta GUI Maker\n    By MarKs\n]]\n\n"
    code = code .. "-- Create the GUI\nlocal gui = Instance.new(\"ScreenGui\")\ngui.Name = \"DeltaGeneratedGUI\"\ngui.ResetOnSpawn = false\ngui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling\n\n"
    
    -- Count elements for variable naming
    local elementCounts = {}
    
    -- Process all elements
    for i, element in ipairs(self.Elements) do
        -- Get element type and count
        local elemType = element.Type
        elementCounts[elemType] = (elementCounts[elemType] or 0) + 1
        local varName = string.lower(elemType) .. elementCounts[elemType]
        
        -- Create element
        code = code .. "local " .. varName .. " = Instance.new(\"" .. elemType .. "\")\n"
        code = code .. varName .. ".Name = \"" .. element.Name .. "\"\n"
        code = code .. varName .. ".Position = " .. self:UDim2ToString(element.Position) .. "\n"
        code = code .. varName .. ".Size = " .. self:UDim2ToString(element.Size) .. "\n"
        code = code .. varName .. ".BackgroundColor3 = " .. self:Color3ToString(element.Color) .. "\n"
        code = code .. varName .. ".BorderSizePixel = 0\n"
        
        -- Add Z-index if not default
        if element.ZIndex and element.ZIndex ~= 1 then
            code = code .. varName .. ".ZIndex = " .. element.ZIndex .. "\n"
        end
        
        -- Add type-specific properties
        if elemType == "TextLabel" or elemType == "TextButton" or elemType == "TextBox" then
            code = code .. varName .. ".Text = \"" .. (element.Text or "") .. "\"\n"
            code = code .. varName .. ".TextColor3 = " .. self:Color3ToString(element.TextColor or Color3.fromRGB(255, 255, 255)) .. "\n"
            code = code .. varName .. ".TextSize = " .. (element.TextSize or 16) .. "\n"
            code = code .. varName .. ".Font = Enum.Font." .. (element.Font and element.Font.Name or "GothamSemibold") .. "\n"
            
            if element.TextWrapped then
                code = code .. varName .. ".TextWrapped = true\n"
            end
            
            if elemType == "TextBox" then
                code = code .. varName .. ".ClearTextOnFocus = " .. (element.ClearTextOnFocus and "true" or "false") .. "\n"
                if element.PlaceholderText then
                    code = code .. varName .. ".PlaceholderText = \"" .. element.PlaceholderText .. "\"\n"
                end
            end
        elseif elemType == "ImageLabel" or elemType == "ImageButton" then
            code = code .. varName .. ".Image = \"" .. (element.Image or "") .. "\"\n"
            code = code .. varName .. ".ScaleType = Enum.ScaleType." .. (element.ScaleType and element.ScaleType.Name or "Stretch") .. "\n"
            
            if element.ImageTransparency and element.ImageTransparency > 0 then
                code = code .. varName .. ".ImageTransparency = " .. element.ImageTransparency .. "\n"
            end
        elseif elemType == "ScrollingFrame" then
            code = code .. varName .. ".CanvasSize = " .. self:UDim2ToString(element.CanvasSize or UDim2.new(0, 0, 2, 0)) .. "\n"
            code = code .. varName .. ".ScrollBarThickness = " .. (element.ScrollBarThickness or 6) .. "\n"
            code = code .. varName .. ".ScrollingDirection = Enum.ScrollingDirection." .. (element.ScrollingDirection and element.ScrollingDirection.Name or "Y") .. "\n"
        end
        
        -- Add corner radius
        code = code .. "\n-- Add rounded corners\n"
        code = code .. "local corner" .. varName .. " = Instance.new(\"UICorner\")\n"
        code = code .. "corner" .. varName .. ".CornerRadius = UDim.new(0, " .. (element.CornerRadius or 8) .. ")\n"
        code = code .. "corner" .. varName .. ".Parent = " .. varName .. "\n"
        
        -- Add script if it exists (for buttons)
        if (elemType == "TextButton" or elemType == "ImageButton") and element.Script and element.Script ~= "" then
            code = code .. "\n-- Add script\n"
            code = code .. "local script" .. varName .. " = Instance.new(\"LocalScript\")\n"
            code = code .. "script" .. varName .. ".Parent = " .. varName .. "\n\n"
            
            -- Add script content with proper indentation
            local scriptContent = element.Script:gsub("\n", "\n    ")
            code = code .. "-- Script content\n"
            code = code .. "script" .. varName .. ".Source = [[\n    " .. scriptContent .. "\n]]\n"
        }
        
        -- Set parent
        code = code .. "\n" .. varName .. ".Parent = gui\n\n"
    end
    
    -- Add final parent to player
    code = code .. "-- Set the GUI parent to PlayerGui\ngui.Parent = game.Players.LocalPlayer:WaitForChild(\"PlayerGui\")\n"
    
    return code
end

-- Helper function to convert UDim2 to string
function DeltaGUIMaker:UDim2ToString(udim2)
    if not udim2 then return "UDim2.new(0, 0, 0, 0)" end
    return string.format("UDim2.new(%.3f, %d, %.3f, %d)", 
        udim2.X.Scale, udim2.X.Offset, 
        udim2.Y.Scale, udim2.Y.Offset)
end

-- Helper function to convert Color3 to string
function DeltaGUIMaker:Color3ToString(color3)
    if not color3 then return "Color3.fromRGB(255, 255, 255)" end
    return string.format("Color3.fromRGB(%d, %d, %d)", 
        math.floor(color3.R * 255), 
        math.floor(color3.G * 255), 
        math.floor(color3.B * 255))
end

-- Initialize and show
local function Start()
    local maker = DeltaGUIMaker.new()
    maker.MainUI.Parent = game.CoreGui
    maker.IsOpen = true
    maker:ShowNotification("Delta GUI Maker loaded!", Color3.fromRGB(80, 255, 120))
end

Start()

return DeltaGUIMaker
